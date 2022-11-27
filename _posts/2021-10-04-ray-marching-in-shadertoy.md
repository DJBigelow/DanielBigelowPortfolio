---
layout: single
title:  "A Tiny Raymarcher in Shadertoy"
date:   2021-10-04 18:40:03 -0600
categories: glsl 3D-graphics
---

# Ray Marching

Rather than explain ray marching myself, I'll defer explanation to the following video, which inspired me to write this project: [https://www.youtube.com/watch?v=svLzmFuSBhk](https://www.youtube.com/watch?v=svLzmFuSBhk)

![Phong Illumination](https://i.imgur.com/hla1TWS.png)

--------------

## The Platform

The following project was programmed in ShaderToy, a platform that allows users to program GLSL-like fragment shaders in the browser. The project could've been written as an OpenGL application, though doing so would've taken a little extra work setting up a rendering loop and applying the shader I wrote to a square mesh encompassing the entire viewport.

--------------

![Gooch Illumination](https://i.imgur.com/NlsrDTK.png)

--------------

## Concepts Demonstrated

Despite ray marching being a non-traditional rendering technique, this project still demonstrates a few concepts fundamental to traditional rasterization rendering, namely models of shading (Gooch and Phong in this instance), rotations and translations (without matrices), and fragment shader programming. Rather than explain how these are achieved in code here, I'll refer you to the code itself, which should have enough comments to give you an idea of what's going on: [https://www.shadertoy.com/view/3l3GR4](https://www.shadertoy.com/view/3l3GR4). Remember to try the different shading models available by changing the constant on line 22!

--------------

![Glow](https://i.imgur.com/ERRTAhn.png)
