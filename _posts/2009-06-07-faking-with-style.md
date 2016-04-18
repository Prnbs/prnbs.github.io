---
layout: post
img: distributed.JPG
title: Faking with Style
tags: graphics
category: projects
summary: How to fake a snapple bottle using a few geometric primitives, a camera and renderman
image: snapple.png
---

### This is faking...with style

The purpose of this project was to create a Snapple bottle entirely from textures.
The first choice that I was faced with was to decide whether to use Renderman or GLSL. I went for the former because of the ease with which textures can be loaded and displacement shaders can be created.

The next thing to do was create a displacement shader which would give the look of a bottle.
My first trial gave me this:

![first trial](/img/snapple/shape.jpg "First trial")

Though it looks like a bottle it is not THE Snapple bottle. Changing the cosine function with a logarithmic function solved the problem for me. Here is the result:

![final shape](/img/snapple/new_shape.jpg "final shape")

Since the intention was to use textures as much as possible I bought a Snapple, removed the label after considerable legedemain and scanned it. From there on it was a simple task of cropping the label 'Snapple' and turning it to grayscale so that it could be used for bump mapping. For the thread of the cap I drew some horizontal lines on a black background and used it for bump mapping.

So here is the image, it looks like a plastic bottle.

![Plastic bottle](/img/snapple/snapplebottleWithoutGlass.png "plastic bottle")

The next step was to render the glass. Now raytracing is surprisingly difficult to do in Renderman, it's like a classified secret and no one on the web talks about it. Since the essence of Computer Graphics can be summed up in Buzz Lightyear's slightly modified words as "This is Faking .....with style", I decided to fake the glass using textures.
After a lot of trials I finally found an image which gave me what I wanted, here is the original image:

![Glass source](/img/snapple/glass_source.jpg "this became the texture for glass")

And this is the result of applying it as a texture, it sort of looks like glass but the illusion is incomplete because it doesn't reflect anything.

![Glass source](/img/snapple/snapplebottleWithOnlyGlass.png "glass bottle but no reflections")

To add reflections  I did environment mapping using the following 180 degree image:

![Env mapping source](/img/snapple/tableenv.png "this image was used to reflect on to the bottle")

And the result of applying this is given below:

![Final image](/img/snapple/final_image.png "final image")

### Volumetric effects

I played around with some more textures and found that given the right image it is also possible to get volume like effects using just textures.
Here are some of the results:

![3 week old soup](/img/snapple/3_week_old_soup.png "3 week old soup")

It reminds me of three week old hot and sour soup

![Smoke on the water](/img/snapple/smoke_on_the_water.png "Smoke on the water")

And I call this one smoke on the water.
