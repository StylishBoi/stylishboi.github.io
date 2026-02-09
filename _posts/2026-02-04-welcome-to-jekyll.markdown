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

When the engine actually starts running, the Begin() function in my Scene class starts and everything gets initialized one by one with a verification each time to avoid errors but make my scene more extensible in different uses (It deeply failed to achieve that however, an effort was more).

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

*Side note, it is essential to set our rendering mode to GL_FRONT to render only back the side of the object as we need to the shadows to form behind the object. If we leave it at GL_BACK, it'll render the front of the object and most of the shadows will be found INSIDE the object.*

**GBuffer**

Our GBuffer will help with doing our deferred lighting, it'll also be essential for our post processing effects (SSAO and Bloom).

Deferred lighting is a more efficient way to calculate the light present in the scene.
Instead of doing the lighting object by object, we'll do the lighting for the whole screen with a single shader which is great in terms of performances but also for future effects that we'll want to add on top of it.

However we cannot do lighting immedietaly, we'll first render our scene geometry inside the gbuffer so it'll be able to store these informations.

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


**Lighting Pass**

**Skybox**

**Lightcubes**

**Bloom/Blur**


Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

**Conclusion**

Creating this scene has opened my eyes to a whole other spectrum of programming which I knew pratically nothing about.

It was simultaneously a really interesting and deeply frustrating experience which challenged a lot of my notions in debugging and architercural design. Would have this been infinitely easier if I didn't attempt to make a barebone engine instead of just a hard-coded scene ? Yes, definitely, it ended up costing me many long hours of work trying to sort out this system I had set up. However, it also taught me a lot about 

Would I be interested in looking further in computer graphics after this ? Undoubtedly, even if I'm not sure to understand everything yet, it is fascinating work that I'll try bettering myself in the future.

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
