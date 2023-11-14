---
title: OpenGLES 3.0 - 着色器语言GLSL
date: 2023-09-27 20:06:30
tags:
    - OpenGL
    - OpenGLES
    - GLSL
categories:
    - OpenGL
---
# 简介
在OpenGL开发过程中，我们经常会用到着色器语言(GLSL)，它主要用于编写顶点着色器和片段着色器，接下来会详细讲解着色器语言GL Shader Language(GLSL)的一些基本概念。

## 版本
GLSL最初随着OpenGL2.0版本一同发布，随着OpenGL版本的不断升级。OpenGL版本和GLSL版本对应关系：
| OpenGL版本 | GLSL版本 | 发布时间 |
| :-----: | :----: | :----: |
| 2.0 | 110 | 2004年9月7日 |
| 2.1 | 120 | 2006年7月2日 |
| 3.0 | 130 | 2008年8月11日|
| 3.1 | 140 | 2009年3月24日|
| 3.2 | 150 | 2009年8月3日 |
| 3.3 | 330 | 2010年3月11日|
| 4.0 | 400 | 2010年3月11日|
| 4.1 | 410 | 2010年7月26日|
| 4.2 | 420 | 2011年8月8日 |
| 4.3 | 430 | 2012年8月6日 |
| 4.4 | 440 | 2013年7月23日|
| 4.5 | 450 | 2014年7月    |
| 4.6 | 460 | 2017年7月|

同样的，对于嵌入式设备，对于OpenGLES，也有对应的GLSL ES版本，对应关系：
| OpenGLES版本 | GLSL ES版本 |
| :-----: | :----: |
| 2.0 | 110 | 
| 3.0 | 130 | 
| 3.1 | 140 |
| 3.2 | 150 | 

本文主要讲解的是OpenGL ES3.0版本的GLSL。

# 变量类型
OpenGL ES 着色器语言支持的变量类型：
| 分类 | 变量类型	 | 描述 |
| :-----: | :----: | :----: |
| 标量    | float,int,bool,uint | 标量也被称为 “无向量”，</br>其值只有大小，并不具有方向 |
| 浮点向量    | float, vec2, vec3, vec4	  |有 1、2、3、4 个分量，</br>基于浮点的向量类型 |
| 正数向量    | int, ivec2, ivec3, ivec4  |有 1、2、3、4 个分量，</br>基于整数的向量类型 |
| 布尔向量    | bool, bvec2, bvec3, bvec4 |有 1、2、3、4 个分量，</br>基于布尔的向量类型 |
| 无符号整数向量    | uint, uvec2, uvec3, uvec4	|有 1、2、3、4 个分量，</br>基于无符号整数的向量类型 |
| 矩阵    | mat2x2,mat2x3,max2x4...| 矩阵是按照列的顺序组织的，</br> 第一个数字表示列数</br>第二个表示行数|
| 纹理句柄  | sampler2D, samplerCube | 表示2D，立方体纹理的句柄 |
| 空类型| void| 用于无返回值的函数或空的</br>参数列表| 

# 结构体
类似C语言中的结构体，使用`struct`关键字声明结构体。
``` glsl
struct foo{
    vec3 position;
    float start;
} fooVar;

fooVar = foo(vec3(1.0,1.0,0.0),0.5);
```
# 数组
可以声明各种类型的数组。
声明数组的方式：
* 声明数组的同时，指定大小：
``` glsl
vec4 position[10];
```

* 声明数组并初始化：
``` glsl
float a[] = float[](1.0,2.0f);
float b[] = float[2](1.0,2.0f);
```
# 函数
函数的声明和C语言类似，从`main`主函数开始执行。  
注意：函数不能够递归调用，且必须声明返回值类型(无返回值时声明为void)
``` glsl
out vec4 fragColor;
void main(){
    vec3 n;
    vec3 l;
    vec4 b;
    ...
    fragColor = diffuse(n,l,b);
    ...
}

vec4 diffuse(vec3 normal, vec3 light, vec4 baseColor) {
    return baseColor * dot(normal, light);
}
```
## 内置函数
OpenGL ES 着色语言有许多内置函数，处理各种计算任务。  
这些内置函数，可分为4种变体：

| 变体类型 | 描述 |
| :-----: | :----: |
| genType  | float,vec2,vec3,vec4	 | 
| genIType | int,ivec2,ivec3,ivec4	 | 
| genBType | uint,uvec2,uvec3,uvec4  |
| genUType | uint,uvec2,uvec3,uvec4  | 

和上文中提到的4种变量对应(float,int,bool,uint)。  
内置函数都是使用了非常高效的方式来实现，必须熟悉一些常见的内置函数。 
# 限定符
## 存储限定符
在GLSL140版本之前，模型的一次渲染过程，顶点着色器中每次运行都会改变的数据称为attribute，每次运行保持不变的数据称为uniform，着色器之间顺序传递的数据称为varying，这些就是所谓的存储限定符。  
![](s_1.png)
为了强化渲染管线的输入输出概念，GLSL140版本后，存储限定符attribute和varying不再被推荐使用，取而代之的是in和out。
![](s_2.png)

### in限定符
in限定符修饰的全局变量，叫输入变量。  
顶点着色器的输入变量只能使用in限定符来修饰。片段着色器可以使用in或者centroid in 限定符来修饰全局变量。  
in限定修饰符和OpenGL ES2.0中的`attribute`限定符类似。
### out限定符
out限定符修饰的全局变量，叫输出变量。  
顶点着色器的输出变量可以使用in或者centroid out限定符来修饰全局变量，用于向渲染管线后继阶段传递当前顶点的数据。  
片段着色器的输出变量只能使用in限定符来修饰，表示由片段着色器写入计算完成片段颜色值的变量。 
### uniform限定符
uniform限定符修饰的全局变量，叫统一变量。统一变量的命名空间在顶点着色器和片段着色器中都是共享的。 
### const限定符
const限定符，可将变量声明为常量，常量只可读，不可修改。常量必须在声明时初始化。例如：
``` glsl
const float PI = 3.14159;
```
### 插值限定符
插值限定符，其主要用于控制顶点着色器传递到片段着色器数据的插值方式。 
| 插值限定符 | 描述 |
| :-----: | :----: |
| smooth  | 默认值，平滑着色，顶点着色器的输出变量在图元中线性插值| 
| flat| 平面着色，将一个顶点视为驱动顶点（取决于图元类型），</br>该顶点的值用于图元中所有片段|
| centroid | 质心采样，使用多重采样渲染时，</br>该限定符可用于强制插值发生在被渲染图元内部，否则图元边缘可能出现伪像|

### layout限定符
layout限定符是OpenGL ES3.3中新增的特性，主要用于设置变量的存储索引值。  
声明的方式：
* 作为接口块定义的一部分或者接口块的成员/仅仅修饰uniform，用于建立其他一致变量声明的参照

``` glsl
<layout 限定符> uniform
```

* 用于修饰被接口限定符修饰的单独变量

``` glsl
<layout 限定符> <接口限定符> <变量声明>
```
使用layout为in变量指定存储索引值：
``` glsl
layout(location = 0) in vec4 vPosition;
layout(location = 1) in vec4 vTexCoord;
```
如果变量是一个数组，则数组的每一个元素都会占据一个location的编号位置，例子中长度为3的数组，会占用5,6,7三个索引位置：
``` glsl
layout(location = 5) uniform mat4 mMatrix[3];
```
对于out变量，layout索引独立于in变量：
``` glsl
layout(location = 0) out vec4 fragColor;
```

### 统一变量块
统一变量块是OpenGL ES3.3中新增的特性，使用于`统一变量缓冲区对象`，通过缓冲对象送入渲染管线，以统一变量块的形式批量传送数据比单个传送效率高。基本语法：
```glsl
[<layout 限定符>] unifrom 一致块名称 {<成员变量列表>} [<实例名>]
```
示例：
```glsl
#version es 300
uniform Transform{
	float radius;
	mat4 modelViewMatrix;
	uniform mat3 normalMatrix;
};

layout(location = 0) in vec4 a_position;

void main() {
    gl_Position = modelViewMatrix * a_position;
}
```
### 精度限定符
较低的精度效率更高，较高的精度效果更高。  
精度限定符种类：
* lowp : 低精度
* mediump : 中精度
* highp : 高精度

> <font color='red'> **注意:**</font> 
> 1. 顶点着色器中，如果没有指定默认精度，则int和float默认精度都是highp
> 2. 在片段着色器中，浮点值没有默认精度，必须由开发者声明
# 内置变量
## 顶点着色器中的内置变量
### 内置输入变量
* gl_VertexID：顶点整数索引
* gl_InstanceID：实例 ID
### 内置输出变量
* gl_Position：顶点位置
* gl_PointSize：点大小
## 片段着色器中的内置变量
### 内置输入变量
* gl_FragCoord：片段位置
* gl_FragColor：片段颜色
* gl_FragData：片段颜色缓冲区
### 内置输出变量
* gl_FragDepth：片段深度值

# 参考
* [OpenGL Shading Language](https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language)
* [GLSL的存储限定符和布局限定符](https://bbs.huaweicloud.com/blogs/313360)
* [GLSL-Versions](https://github.com/mattdesl/lwjgl-basics/wiki/GLSL-Versions)
* [Android OpenGL ES - GLSL基础篇](https://segmentfault.com/a/1190000037495091)
* [Android OpenGL ES - GLSL高级篇](https://segmentfault.com/a/1190000037563032)
* [OpenGL ES 3.0 shader 着色语言基础语法](https://blog.csdn.net/afei__/article/details/88922112)