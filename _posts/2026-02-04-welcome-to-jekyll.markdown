---
layout: post
title:  "Computer Graphics Scene"
date:   2026-02-04 14:09:22 +0100
categories: jekyll update
---
During the 2nd year of my Bachelors degree in Games Programming, I was tasked to make a 3D scene using the OpenGL API.
Through this post, I'll detail step by step the inner workings of my scene and the different techniques, functionalities and concepts I've learned throughout all of it.

**Overview**

To start off, it would be a good idea to have a proper plan of the general workings of my scene before heading into the details.

My scene is also particular as most of my code is subdiviced in many classes and subclasses, even the scene itself is actually a class. I'm highlighting this fact since normally, we weren't really supposed to do that and writing most of our code in the main file would have been fine.
However silly me decided to do otherwise which heavily increased my workload and the complexicity of my scene for no real reason apart that it sounded nice to me.
So you'll see throughout this post the many problems that came from that.

**Initialization**

Before my scene can even begin to render, it is essential to initialize all the tools that we'll be using throughout the scene or otherwise, they'll start off empty which simply won't work.

When my renderobjects, lightbox and framebuffers are added into my scene, they are not directly initialized. It has to wait until the engine starts running to be intialized otherwise, they'll have no OpenGL framework to build themselves on.

When the engine actually starts running, the `Begin()` function in my Scene class starts and everything gets initialized one by one with a verification each time to avoid errors but make my scene more extensible in different uses (It deeply failed to achieve that however, an effort was more).

{% highlight ruby %}
void Begin() override {

    //---SET UP FRAMEBUFFERS---
    if (gbuffer) {
      gbuffer->Initialize();
    }
    if (hdrbloom) {
      hdrbloom->SetShareBuffer(gbuffer->GetShareBuffer());
      hdrbloom->Initialize();
    }
    if (ssaorenderer) {
      ssaorenderer->SetCamera(*camera);
      ssaorenderer->SetGBufferTextures(gbuffer);
      ssaorenderer->Initialize();
    }
    if (depthmap) {
      depthmap->Initialize();
    }
    //---SET UP OBJECTS---
    for (auto &obj : objects) {
      obj->Initialize(*camera);
      if (obj->isInstanced) {
        gbuffer->bindGeometryPipeline();
        obj->SetupInstancedAttributes();
      }
    }

    //---SET UP LIGHTS---
    for (auto &light : deferredlights) {
      light->Initialize(*camera);
    }

    //---SET UP SKYBOX---
    if (skybox) {
      skybox->Initialize(*camera);
    }

    //---SET UP LIGHTS INFORMATIONS---
    // Honestly the light themselves should have that information, why need a
    // seperate vector I don't know...
    for (auto *l : deferredlights) {
      pointLights.push_back({l->getPosition(), glm::vec3(0.2f), glm::vec3(1.0f),
                             glm::vec3(1.0f), glm::vec3(1.0f, 1.0f, 1.0f), 1.0f,
                             0.09f, 0.032f});
    }
  }
{% endhighlight %}

**Depth Pass/Shadow Mapping**

Now let's get in the actual rendering of the scene, it all starts off with the depth pass which basically sets the shadows that'll be present in the scene.

How it works is actually quite simple, it will recreate the scene from the perspective of the light and define where the shadows based on that view.

*Side note, it is essential to set my rendering mode to `GL_FRONT` to render only back the side of the object as we need to the shadows to form behind the object. If we leave it at `GL_BACK`, it'll render the front of the object and most of the shadows will be found INSIDE the object.*

![DepthPass]({{ site.baseurl }}/assets/img/DepthPass.png)

**Frustum Culling**

Before we render the objects in the scene, it's important to know which objects are going to be rendered.
Frustum culling helps with that by determing which objects are present in the camera frame, allowing us to only draw them and ignore the ones we don't see.

We'll update each object volumes and see if they are visible or not.
If they are visible, they will be added in the `visibleObjects` vector to be drawn later.
{% highlight ruby %}
std::vector<graphics::RenderObject *> visibleObjects;
    visibleObjects.clear();
    for (auto &obj : objects) {
      obj->UpdateBoundingVolumes();
      if (obj->IsVisible(frustum)) {
        visibleObjects.push_back(obj);
      }
    }
{% endhighlight %}

**GBuffer**

My GBuffer will help with doing my deferred lighting, it'll also be essential for my post processing effects (SSAO and Bloom).

Deferred lighting is a more efficient way to calculate the light present in the scene.
Instead of doing the lighting object by object, we'll do the lighting for the whole screen with a single shader which is great in terms of performances but also for future effects that we'll want to add on top of it.

However we cannot do lighting immedietaly, we'll first render my scene geometry inside the gbuffer so it'll be able to store these informations.

{% highlight ruby %}
void RenderGBufferPass(std::vector<RenderObject *> objects,
                         glm::mat4 projection, glm::mat4 view) {
    glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    geometryPipeline.Bind();
    geometryPipeline.SetMat4("projection", glm::value_ptr(projection));
    geometryPipeline.SetMat4("view", glm::value_ptr(view));

    for (auto &obj : objects) {
      obj->RenderGBuffer(geometryPipeline);
    }

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
  }
{% endhighlight %}

**SSAO**

There's now one last step before we can add lighting and that is SSAO.
Screen Space Ambient Occlusion (SSAO) is a technique in which we simulate the soft shadows such as contact shadows or corners.

In my code, there's two passes for it.
First, we determine on the scene where the soft shadows should be present with the help of the shader.
{% highlight ruby %}
void RenderSSAO() {
    glBindFramebuffer(GL_FRAMEBUFFER, ssaoFramebuffer);
    glClear(GL_COLOR_BUFFER_BIT);

    pipeline.Bind();
    pipeline.SetInt("gPosition", 0);
    pipeline.SetInt("gNormal", 1);
    pipeline.SetInt("texNoise", 2);

    for (unsigned int i = 0; i < 64; ++i) {
      pipeline.SetVec3("samples[" + std::to_string(i) + "]", ssaoKernel[i]);
    }
    pipeline.SetMat4("projection", glm::value_ptr(camera->GetProjection()));
    pipeline.SetMat4("view", glm::value_ptr(camera->GetViewMatrix()));
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, main_gPosition);

    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, main_gNormal);

    glActiveTexture(GL_TEXTURE2);
    glBindTexture(GL_TEXTURE_2D, noiseTexture);

    vertexInput.Bind();
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
  }
{% endhighlight %}

![SSAORender]({{ site.baseurl }}/assets/img/SSAO.png)

Then with these shadows, we'll add a blur on them to avoid blockiness that could be present on them.
{% highlight ruby %}
void RenderSSAOBlur() {
    glBindFramebuffer(GL_FRAMEBUFFER, ssaoBlurFramebuffer);
    glClear(GL_COLOR_BUFFER_BIT);
    ssaoBlurPipeline.Bind();

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, ssaoColorBuffer);

    vertexInput.Bind();
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
  }
{% endhighlight %}

![SSAOBlur]({{ site.baseurl }}/assets/img/SSAOBlur.png)

**Lighting Pass**

Now that we have shadows, lighting and SSAO, we can finally all put it together to render my scene.

In my GBuffer again, we'll go through the LightingPass function and use the informations gotten from the DepthPass and SSAO to apply it on the final result. 

{% highlight ruby %}
void LightingPass(std::vector<PointLight> pointLights,
                    const glm::vec3 &viewPos, HDRBloom *hdrbloom,
                    GLuint ssaotexture, GLuint shadowMap,
                    const glm::mat4 &lightSpaceMatrix) {

    if (hdrbloom) {
      hdrbloom->BindForLighting();
      glClear(GL_COLOR_BUFFER_BIT);
    } else {
      glBindFramebuffer(GL_FRAMEBUFFER, 0);
      glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    }

    lightingPipeline.Bind();
    vertexInput.Bind();

    lightingPipeline.SetInt("gPosition", 0);
    lightingPipeline.SetInt("gNormal", 1);
    lightingPipeline.SetInt("gAlbedoSpec", 2);
    lightingPipeline.SetInt("ssao", 3);

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, gPosition);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, gNormal);
    glActiveTexture(GL_TEXTURE2);
    glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
    glActiveTexture(GL_TEXTURE3);
    glBindTexture(GL_TEXTURE_2D, ssaotexture);

    lightingPipeline.SetInt("shadowMap", 5);
    glActiveTexture(GL_TEXTURE5);
    glBindTexture(GL_TEXTURE_2D, shadowMap);

    lightingPipeline.SetMat4("lightSpaceMatrix",
                             glm::value_ptr(lightSpaceMatrix));

    for (unsigned int i = 0; i < pointLights.size(); i++) {
      lightingPipeline.SetVec3("lights[" + std::to_string(i) + "].Position",
                               pointLights[i].position);
      lightingPipeline.SetVec3("lights[" + std::to_string(i) + "].Color",
                               pointLights[i].color);
      // update attenuation parameters and calculate radius
      const float constant = 1.0f;
      const float linear = 0.07f;
      const float quadratic = 0.3f;
      lightingPipeline.SetFloat(
          ("lights[" + std::to_string(i) + "].Linear").c_str(), linear);
      lightingPipeline.SetFloat(
          ("lights[" + std::to_string(i) + "].Quadratic").c_str(), quadratic);
      // then calculate radius of light volume/sphere
      const float maxBrightness =
          std::fmaxf(std::fmaxf(pointLights[i].color.r, pointLights[i].color.g),
                     pointLights[i].color.b);
      float radius =
          (-linear +
           std::sqrt(linear * linear -
                     4 * quadratic *
                         (constant - (256.0f / 1.0f) * maxBrightness))) /
          (2.0f * quadratic);
      lightingPipeline.SetFloat(
          ("lights[" + std::to_string(i) + "].Radius").c_str(), radius);
    }
    lightingPipeline.SetVec3("viewPos", viewPos);

    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

    glBindFramebuffer(GL_READ_FRAMEBUFFER, gBuffer);
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0); // write to default gbuffer
    glBlitFramebuffer(0, 0, common::GetWindowSize().x,
                      common::GetWindowSize().y, 0, 0,
                      common::GetWindowSize().x, common::GetWindowSize().y,
                      GL_DEPTH_BUFFER_BIT, GL_NEAREST);
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
  }
{% endhighlight %}

![LightingPass]({{ site.baseurl }}/assets/img/LightingPass.png)

**Cubemap**

The skybox/cubemap will be rendered after the lighting pass as it is not supposed to be affected by lighting in my case.
{% highlight ruby %}
void Render() override {
    glDepthFunc(GL_LEQUAL);

    pipeline.Bind();

    glm::mat4 view = glm::mat4(glm::mat3(camera->GetViewMatrix()));
    pipeline.SetMat4("view", glm::value_ptr(view));
    pipeline.SetMat4("projection", glm::value_ptr(camera->GetProjection()));

    vertexInput.Bind();

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);
    glDrawElements(GL_TRIANGLES, 36, GL_UNSIGNED_INT, nullptr);

    glDepthFunc(GL_LESS);
  }
{% endhighlight %}

**Lightcubes**

The same thing is also true for my lightcubes, as they're a visual representation for my lighting sources, they aren't affected by shadows however their presence is still important for the final step of my program.

**Bloom/Blur**

For the final part of my scene, we'll have to go through a blooma after affect which will add a sense of bluriness to the lights via a ping pong blur.
How it works is that the Bloom Render will take the our scene and increase the bluriness on the light sources, after that, it'll apply the final result on the screen.
Also marking the last step before rendering our screen.

**Conclusion**

Creating this scene has opened my eyes to a whole other spectrum of programming which I knew pratically nothing about.

It was simultaneously a really interesting and deeply frustrating experience which challenged a lot of my notions in debugging and architercural design. Would have this been infinitely easier if I didn't attempt to make a barebone engine instead of just a hard-coded scene ? Yes, definitely, it ended up costing me many long hours of work trying to sort out this system I had set up. However, it also taught me a lot about 

Would I be interested in looking further in computer graphics after this ? Undoubtedly, even if I'm not sure to understand everything yet, it is fascinating work that I'll try bettering myself in the future.

{% assign image_files = site.static_files | where: "image", true %}
{% for myimage in image_files %}
  {{ myimage.path }}
{% endfor %}

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
