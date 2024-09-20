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
在前面几个章节中，我们使用到了很多概念，没有进行描述，本节针对前面使用到的一些概念，再进行一个普及。已经涉及到的几个概念：VBO、EBO(IBO)、VAO。  
VBO和EBO是在OpenGL1.5中引入的，主要用于优化顶点数据的存储和索引管理。   
VAO则是在OpenGL3.0中引入的，用于简化顶点缓冲和属性的管理。 
VBO和EBO的作用是在显存中申请一块内存，用于缓存数据，降低内存拷贝带来的开销。之前用于绘制的顶点数据会先保存在CPU，每次调用绘制函数的时候，再把顶点数据从CPU拷贝到显存中，这也是一个比较大的开销。
<!-- more -->
# VBO
VBO(Vertex Buffer Object)，顶点缓存对象。它将数据(顶点位置、颜色、纹理坐标等)存储到GPU显存中。
主要流程:
* 创建一个VBO对象
* 绑定VBO
* 调用glBufferData函数，将数据拷贝到显存中
* 链接顶点属性

示例代码:
``` c++
//1. create vbo
glGenBuffers(1, &mVBO);

//2. bind vbo
glBindBuffer(GL_ARRAY_BUFFER, mVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(mVertices), mVertices, GL_STATIC_DRAW);

//3. set vertex attribute
//xyz
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (void *) 0);
glEnableVertexAttribArray(0);
//rgb
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat),
                        (void *) (3 * sizeof(GLfloat)));
glEnableVertexAttribArray(1);

//st
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat),
                        (void *) (6 * sizeof(GLfloat)));
glEnableVertexAttribArray(2);
```
## 链接顶点属性
使用到的顶点数据：
``` c++
GLfloat mVertices[] = {
        //xyz, rgb, st
        0.5f, 0.5f, 0.0f, 1.0f, 0.0f, 0.0f, 1.0f, 1.0f,
        0.5f, -0.5f, 0.0f, 0.0f, 1.0f, 0.0f, 1.0f, 0.0f,
        -0.5f, -0.5f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f, 0.0f,
        -0.5f, 0.5f, 0.0f, 1.0f, 1.0f, 0.0f, 0.0f, 1.0f
};
```
在绘制渲染前，需要指定OpenGL该如何解析顶点数据，使用`glVertexAttribPointer`函数进行解析。
![](vertex_attribute_pointer_interleaved_textures.png)
``` c++
//2.rgb
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat),
                        (void *) (3 * sizeof(GLfloat)));
glEnableVertexAttribArray(1);
```
`glVertexAttribPointer`参数：
1. 配置的顶点属性。顶点着色器中使用`layout(location = 0)`定义了position顶点属性的位置值
2. 顶点属性的大小。顶点属性是`vec3`，值是3
3. 指定数据的类型。`GL_FLOAT`(GLSL中vec*都是由浮点数值组成的)
4. 数据是否被标准化(Normalize)。`GL_TRUE`，数据都会被映射到0到1之间。这里设置为`GL_FALSE`
5. 步长(stride)。连续的顶点属性组之间的间隔。下个顶点是在8个`float`后，步长设置为`8*sizeof(float)`
6. 起始位置的偏移量(Offset)。颜色的偏移量是在位置数据(3个`float`)后，所以偏移量是`3*sizeof(float)`
# EBO
VBO(Vertex Buffer Object)，元素缓冲对象。用于存储顶点索引的缓冲对象，通过索引来引用顶点数据，避免了重复存储相同的顶点数据。EBO 可以与 VBO 结合使用，实现更高效的渲染。
主要流程：
* 创建一个EBO对象
* 绑定EBO
* 调用glBufferData函数，将数据拷贝到显存中
* 绘制：使用 glDrawElements() 按照索引顺序绘制图形

示例代码:
``` c++
//1. create ebo
glGenBuffers(1, &mEBO);

//2. bind ebo
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, mEBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(mIndices), mIndices, GL_STATIC_DRAW);

//3. draw
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, (void *) 0);
``` 
# VAO
VAO(Vertex Array Object，顶点数组对象)，元素缓冲对象，用于存储顶点属性配置状态的对象。减少`glBindBuffer`、`glEnableVertexAttribArray`、 `glVertexAttribPointer`的调用，简化了与VBO和EBO相关的状态管理。

VAO、VBO、EBO之间的关系：
![](vertex_array_objects_ebo.png)
主要流程：
1. 创建一个VAO对象
2. 绑定VAO
3. 配置VBO和EBO
4. 绘制：当需要渲染时，直接绑定VAO，再进行绘制即可

示例代码：
```c++
//1. create vao
glGenVertexArrays(1, &mVAO);
//2. bind vao
glBindVertexArray(mVAO);

//3. record vbo and ebo
glBindBuffer(GL_ARRAY_BUFFER, mVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(mVertices), mVertices, GL_STATIC_DRAW);

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, mEBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(mIndices), mIndices, GL_STATIC_DRAW);
//xyz
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (void *) 0);
glEnableVertexAttribArray(0);
//rgb
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat),
                        (void *) (3 * sizeof(GLfloat)));
glEnableVertexAttribArray(1);
//st
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat),
                        (void *) (6 * sizeof(GLfloat)));
glEnableVertexAttribArray(2);

//4. draw
glBindVertexArray(mVAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, (void *) 0);
```
通过VAO，你只需设置一次 VBO、EBO和顶点属性，之后可以快速切换不同的VAO进行渲染，避免重复配置，提高了开发效率和渲染性能。
在面试中经常被问到的问题，VBO、EBO和VAO中哪些是可共享的资源，正好一起梳理下：
* 可以共享的资源：
> 纹理
shader
program着色器程序
buffer类对象，如VBO、EBO、RBO等
* 不可以共享的资源：
> FBO帧缓冲区对象（不属于buffer类）
VAO顶点数组对象（不属于buffer类）

# 参考
* [关于OpenGL的渲染上下文](https://cloud.tencent.com/developer/article/2357696)
* [三角形](https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/)
* [OpenGL History](https://www.khronos.org/opengl/wiki/History_of_OpenGL)