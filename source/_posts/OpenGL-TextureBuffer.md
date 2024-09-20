---
title: OpenGLES 3.0 - VBO、EBO和VAO
date: 2023-10-09 11:41:11
tags:
    - OpenGL
    - OpenGLES
    - Texture
categories:
    - OpenGL
---



可以共享的资源：
纹理；
shader；
program 着色器程序；
buffer 类对象，如 VBO、 EBO、 RBO 等 。

不可以共享的资源：
FBO 帧缓冲区对象（不属于 buffer 类）；
VAO 顶点数组对象（不属于 buffer 类）。

# 参考
* [关于 OpenGL 的渲染上下文](https://cloud.tencent.com/developer/article/2357696)
* [三角形](https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/)