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

    if (gbuffer) {
      gbuffer->Initialize();
    }
    //---SET UP FRAMEBUFFERS---
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
      if(obj->isInstanced){
        gbuffer->bindGeometryPipeline();
        obj->SetupInstancedAttributes();
      }
    }

    for (auto &light : deferredlights) {
      light->Initialize(*camera);
    }

    //---SET UP SKYBOX---
    if (skybox) {
      skybox->Initialize(*camera);
    }

    //---SET UP LIGHTS INFORMATIONS---
    for (auto *l : deferredlights) {
      pointLights.push_back({l->getPosition(), glm::vec3(0.2f), glm::vec3(1.0f),
                             glm::vec3(1.0f), glm::vec3(1.0f, 1.0f, 1.0f), 1.0f,
                             0.09f, 0.032f});
    }
  }
{% endhighlight %}

**Depth Pass/Shadow Mapping**

**GBuffer**

**SSAO**

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
