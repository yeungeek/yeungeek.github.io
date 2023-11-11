---
title: OpenGLES 3.0 - 简介和基础绘制流程
date: 2023-08-02 11:15:53
tags:
    - OpenGL
    - OpenGLES
categories:
    - OpenGL
---
# 简介
## OpenGL
![pPK8x4P.jpg](https://s1.ax1x.com/2023/08/13/pPK8x4P.jpg)
[OpenGL(Open Graphics Library,开放图形库)](https://www.opengl.org/)，是用于渲染2D、3D矢量图形的跨语言、跨平台的应用程序编程接口规范。 
它本身并不是一个API，它仅仅是一个由Khronos组织制定并维护的规范（Specification）。它的实现由显示设备厂商提供，并且非常依赖该厂商提供的硬件。  
当你使用Apple系统的时候，OpenGL库是由Apple维护的。在Windows上，OpenGL库可能是Direct3D接口的一个封装(Wrapper)。表面上调用OpenGL接口，实际上是间接调用了Direct3D接口。
<!--more-->
像现在非常火爆的GPU生产商Nvidia，会提供满足OpenGL规范的实现，它们负责将OpenGL定义的API命令翻译为GPU指令。  
OpenGL应用场景广泛，通常用于CAD、科学可视化、游戏开发等，像游戏开发引擎Unity,Unreal底层渲染引擎都是基于OpenGL，当然也支持Vulkan。
## OpenGL ES
![](https://s1.ax1x.com/2023/08/13/pPKULDS.png)
[OpenGL ES(OpenGL for Embedded Systems)](https://www.khronos.org/opengles)，是 OpenGL 三维图形API的子集，针对手机、PDA和游戏主机等嵌入式设备而设计，去除了许多不必要和性能较低的API接口。本系列文章介绍OpenGL版本基于OpenGL ES3.0， 是OpenGLES 2.0的扩展版本，向下兼容OpenGLES 2.0 ，但不兼容OpenGLES 1.0。

### 3.0主要新特性
#### 纹理
* sRGB 纹理和帧缓冲区 - 允许应用程序执行伽玛校正渲染
* 2D纹理数组 - 存储2D纹理数组的纹理目标
* 3D纹理 - OpenGL ES 3.0中作为强制功能
* 深度纹理和阴影比较 - 允许将深度缓冲区存储在纹理中
* 无缝立方体贴图 - 在OpenGL ES 2.0中，使用立方体贴图进行渲染可能会在立方体贴图面之间的边界处产生伪影。在OpenGL ES 3.0中，可以对立方体贴图进行采样，以便过滤使用来自相邻面的数据并消除接缝伪影
* 浮点纹理 - OpenGL ES 3.0极大地扩展了支持的纹理格式
* ETC2/EAC 纹理压缩
* 整数纹理 - OpenGL ES 3.0引入了渲染和提取存储为非标准化有符号或无符号8位、16位和32位整数纹理的纹理的功能
* 其他纹理格式
* 纹理混合 - 引入了新的纹理对象状态，以允许独立控制纹理数据的每个通道（R、G、B 和 A）在着色器中映射到的位置
* 不可变纹理 — 为应用程序提供一种机制，在加载数据之前指定纹理的格式和大小
* 增加了最小尺寸 - OpenGL ES 3.0 最小2D纹理尺寸为2048

#### 着色器
* 程序二进制文件 - 在 OpenGL ES 3.0中，整个链接程序二进制文件（包含顶点和片段着色器）可以以离线二进制格式存储，运行时无需链接步骤
* 强制在线编译器
* 非方阵 - 支持方阵以外的新矩阵类型，并且将关联的统一调用添加到API中以支持加载它们
* 完整整数支持
* 质心采样
* 平面/平滑插值程序
* 统一变量块 - 统一变量值可以组合为统一变量块。统一变量块可以更高效地加载，也可在多个着色器程序间共享
* 布局限定符
* 实例和顶点ID
* 片段深度 - 片段着色器可以显式控制当前片段的深度值，而不是依赖于其深度值的插值
* 新的内置函数
* 放宽限制 - OpenGL ES 3.0放宽了对着色器的限制。着色器不再受指令长度限制，完全支持变量循环和分支，并支持数组索引

#### 几何形状
* 变换反馈 - 允许在缓冲区对象中捕获顶点着色器的输出
* 布尔遮挡查询
* 实例化渲染 - 高效渲染包含相似几何图形但属性（例如变换矩阵、颜色或大小）不同的对象
* 图元重新启动
* 新的顶点格式 - 新的顶点格式，包括10-10-10-2有符号和无符号标准化顶点属性

#### 缓冲区对象
* 统一缓冲区对象 - 提供存储/绑定大块统一的有效方法
* 顶点数组对象 - 提供一种有效的方法来绑定和切换顶点数组状态
* 采样器对象 - 将采样器状态（纹理环绕模式和过滤）与纹理对象分开
* 同步对象 - 为应用程序提供一种机制来检查一组 OpenGL ES 操作是否已在GPU上完成执行
* 像素缓冲区对象 - 使应用程序能够执行数据到像素操作和纹理传输操作的异步传输
* 缓冲区子范围映射 - 允许应用程序映射缓冲区的子区域以供CPU访问
* 缓冲区对象间拷贝

#### 帧缓冲区
* 多个渲染目标(MRT) — 允许应用程序一次同时渲染到多个颜色缓冲区
* 多重采样渲染缓冲区 - 使应用程序能够通过多重采样抗锯齿渲染到离屏帧缓冲区
* 帧缓冲区失效提示 
* 新的混合方程 - OpenGL ES 3.0支持最小/最大函数作为混合方程

# OpenGL绘制流程
本系列是基于Android的OpenGL，绘制使用了GLSurfaceView，具体的渲染实现使用JNI实现。  
使用GLSurfaceView，需要设置一个Renderer，整个渲染流程是由Renderer实现，主要实现三个方法：
``` java
open class ShaderRenderer(sample: Sample) : Renderer {
    private val nativeRender: ShaderNativeRender
    private val mSample:Sample
    init {
        nativeRender = ShaderNativeRender()
        mSample = sample
    }

    override fun onSurfaceCreated(gl: GL10?, config: EGLConfig?) {
        // GLSurfaceView视图创建时候调用
        nativeRender.native_OnSurfaceCreated()
    }

    override fun onSurfaceChanged(gl: GL10?, width: Int, height: Int) {
        // GLSurfaceView视图改变时调用
        nativeRender.native_OnSurfaceChanged(width,height)
    }

    override fun onDrawFrame(gl: GL10?) {
        // 每帧调用
        nativeRender.native_OnDrawFrame()
    }

    fun onCreate() {
        nativeRender.native_Init(mSample.id)
    }

    fun onDestroy() {
        nativeRender.native_UnInit()
    }
}
```
调用Native，JNI的声明:
``` java
class ShaderNativeRender() {
    companion object {
        init {
            System.loadLibrary("shader-render")
        }
    }

    external fun native_Init(id:Int)

    external fun native_UnInit()

    external fun native_OnSurfaceCreated()

    external fun native_OnSurfaceChanged(width: Int, height: Int)

    external fun native_OnDrawFrame()
}
```
JNI层的部分实现：
``` C++
void ShaderContext::OnSurfaceCreated() {
    LOGD("###### ShaderContext OnSurfaceCreated");
    glClearColor(1.0f, 1.0f, 1.0f, 1.0f);

}

void ShaderContext::OnSurfaceChanged(int width, int height) {
    LOGD("###### ShaderContext OnSurfaceChanged,w=%d,h=%d", width, height);
    glViewport(0, 0, width, height);
}

void ShaderContext::OnDrawFrame() {
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
}
```

## 渲染管线
学习OpenGL，渲染流程是非常重要的，下图是图形渲染管线每个阶段的抽象展示
![](pipeline.png)
开始绘制之前，需要提供给OpenGL一些顶点数据，作为顶点输入，接下来就是图形渲染管线的第一部分 - 顶点着色器。
### 顶点着色器
顶点着色器(Vertex Shader)是几个可编程着色器中的一个，主要作用是`确定绘制图形的形状`。  
定义一个着色器程序，需要通过GLSL(OpenGL Shading Language)语言来编写，下面这段是非常典型的顶点着色器源码：
``` glsl
#version 330 core
layout (location = 0) in vec3 aPos;

void main()
{
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```
使用`in`关键字定义输入顶点属性，把位置数据赋值给预定义的`gl_Position`变量，设置的值会成为该顶点着色器的输出。
### 图元装配
输入的数据，就是上一个阶段顶点着色器输出的所有顶点，并将输入的数据装配成指定图元的形状。图元的类型：点、线、三角形，其他复杂的图形都是由这三种基本的图形组成。
### 几何着色器
几何着色器把图元形式的一系列顶点的集合作为输入，通过对输入的顶点进行处理，会决定输出的图元类型和个数。
### 光栅化
这个阶段会把图元映射为最终屏幕上的像素，生成供片段着色器(Fragment Shader)使用的片段(Fragment)。
### 片段着色器
主要目的是计算一个像素的最终颜色，这是另外一个可编程的着色器。
### 测试与混合
这个阶段检测片段的深度，判断像素和其他物体的前后关系，确定是否丢弃，检查Alpha值并对物体进行混合。

可以看到渲染管线是非常复杂的，包含了很多可配置的部分。其中顶点着色器和片段着色器是可编程，也是我们需要关注的。几何着色器是可选，一般使用默认即可。
## 代码流程
### 定义着色器
顶点着色器：
``` glsl
#version 300 es
layout (location = 0) in vec4 vPosition;

void main()
{
    gl_Position = vPosition;
}
```
片段着色器：
```glsl
#version 300 es
precision mediump float;
out vec4 fragColor;
void main()
{
    fragColor = vec4(1.0,0.0,0.0,1.0);
}

```
### 编译OpenGL程序
根据顶点和片段着色器类型，创建和编译着色器程序
``` C++
//根据不同的类型创建着色器ID
GLuint shader = glCreateShader(shaderType);
if (shader) {
    //将着色器ID和内容连接
    glShaderSource(shader, 1, &source, NULL);
    //编译着色器
    glCompileShader(shader);

    GLint compiled = 0;
    //检查是否编译成功
    glGetShaderiv(shader, GL_COMPILE_STATUS, &compiled);
    if (!compiled) {
        GLint infoLen = 0;
        glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLen);
        if (infoLen) {
            char *buf = (char *) malloc((size_t) infoLen);
            if (buf) {
                //获取错误消息
                glGetShaderInfoLog(shader, infoLen, NULL, buf);
                LOGE("###### LoadShader compiled error %d, \n%s", shaderType, buf);
                free(buf);
            }
            glDeleteShader(shader);
            shader = 0;
        }
    }
}
```
* GLuint glCreateShader(GLenum shaderType)：根据不同的类型创建着色器ID
* void glShaderSource(GLuint shader, GLsizei count, const GLchar **string, const GLint *length)：
将着色器ID和内容连接
* void glCompileShader(GLuint shader)：编译着色器
* void glGetShaderiv(GLuint shader, GLenum pname, GLint *params)：检查编译是否成功
* void glGetShaderInfoLog(	GLuint shader, GLsizei maxLength, GLsizei *length, GLchar *infoLog)：
获取错误消息
### 创建OpenGL程序和链接着色器
接下来就是创建OpenGL程序，并链接着色器
``` C++
program = glCreateProgram();
LOGD("###### Create Program Result: %d", program);
if (program) {
    glAttachShader(program, vertexShaderId);
    CheckGLError("glAttachShader");
    glAttachShader(program, fragmentShaderId);
    CheckGLError("glAttachShader");
    //链接OpenGL程序
    glLinkProgram(program);
    GLint linkResult = GL_FALSE;
    glGetProgramiv(program, GL_LINK_STATUS, &linkResult);
...
}
```
* GLuint glCreateProgram(void)：创建程序
* void glAttachShader(GLuint program, GLuint shader)：将指定的shader附着到指定的program对象上
* void glLinkProgram(GLuint program)：链接OpenGL程序

### 使用OpenGL程序
接着调用`glUseProgram`，创建的程序对象作为它的参数，之后每个着色器调用和渲染调用都会使用这个程序对象。
* void glUseProgram(GLuint program)：使用OpenGL程序
### 绘制
最后一步就是绘制流程：
``` C++
...
if (mProgram == 0) {
    return;
}

//clear
glClear(GL_COLOR_BUFFER_BIT);

//load vertex data
//设置顶点属性数据
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, mVertices);
//启用顶点属性
glEnableVertexAttribArray(0);

//draw
glDrawArrays(GL_TRIANGLES, 0, 3);

glDisableVertexAttribArray(0);
...
```
* void glVertexAttribPointer(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void *pointer)：
定义通用顶点属性数据
    * index：配置的顶点属性，opengl3.0在顶点着色器定义的`layout(location=0)`表示位置为0
    * size：顶点属性的大小，顶点属性vec3，这个值就是3
    * type：数据的类型，`GL_FLOAT`表示是浮点数值
    * normalized：是否归一化
    * stride：步长，连续的顶点属性组之间的间隔。设置为0，则是让OpenGL决定具体的步长
    * pointer：表示位置数据在缓冲中起始位置的偏移量(Offset)
* void glEnableVertexAttribArray(GLuint index)：启用通用顶点属性数组，允许顶点着色器读取GPU数据
* void glDrawArrays(GLenum mode, GLint first, GLsizei count)：绘制图形
    * mode：绘制类型，包含`GL_POINTS`、`GL_LENS`、`GL_TRIANGLE`等

![](draw_mode.jpeg)
这个就是OpenGL的整体绘制流程，遵循了GPU的渲染管线流程。

# 参考
* [OpenGL](https://www.opengl.org/)
* [OpenGL ES](https://www.khronos.org/opengles/)
* [OpenGL Refpages](https://registry.khronos.org/OpenGL-Refpages/gl4/)
* [Learn OpenGL](https://learnopengl-cn.github.io/)
* [一看就懂的OpenGL ES教程——图形渲染管线的那些事](https://juejin.cn/post/7119135465302654984)