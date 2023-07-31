---
title: Android Camera-相机尺寸、方向和图像数据
date: 2020-01-24 00:14:31
tags:
   - Camera
   - Camera2
   - Orientation
categories:
   - Camera 
---
前面几篇文章介绍了Camera1，Camera2，CameraView和CameraX的使用，对各个API的使用，应该问题不大，不过在真正开发过程中，也会遇到各种不同的问题，本篇文章继续介绍相机开发过程中遇到的问题，主要是相机预览、拍照尺寸，方向，以及图像数据的处理。
<!--more-->
# 尺寸
这里的尺寸，主要是预览尺寸、拍照尺寸和显示预览画面的View大小。  
## 预览尺寸
如何获取预览尺寸?我们可以从[cameraview](https://github.com/google/cameraview)的源码中获取到，分为了Camera1和Camera2。   
### Camera1
``` java
mCameraParameters = mCamera.getParameters();
// Supported preview sizes
mPreviewSizes.clear();
for (Camera.Size size : mCameraParameters.getSupportedPreviewSizes()) {
    Log.d("DEBUG", "###### SupportedPreviewSizes: width=" + size.width + ", height="
            + size.height);
    mPreviewSizes.add(new Size(size.width, size.height));
}
```
### Camera2
``` java
mPreviewSizes.clear();
for (android.util.Size size : map.getOutputSizes(mPreview.getOutputClass())) {
    int width = size.getWidth();
    int height = size.getHeight();
    if (width <= MAX_PREVIEW_WIDTH && height <= MAX_PREVIEW_HEIGHT) {
        mPreviewSizes.add(new Size(width, height));
    }
}
```
不同的厂商和系统所支持的预览尺寸是不一样，下面是红米Note 5A手机上支持的所有预览尺寸：
``` java
SupportedPreviewSizes: width=1280, height=720
SupportedPreviewSizes: width=960, height=720
SupportedPreviewSizes: width=864, height=480
SupportedPreviewSizes: width=800, height=480
SupportedPreviewSizes: width=768, height=432
SupportedPreviewSizes: width=720, height=480
SupportedPreviewSizes: width=640, height=640
SupportedPreviewSizes: width=640, height=480
SupportedPreviewSizes: width=480, height=640
SupportedPreviewSizes: width=640, height=360
SupportedPreviewSizes: width=576, height=432
SupportedPreviewSizes: width=480, height=360
SupportedPreviewSizes: width=480, height=320
SupportedPreviewSizes: width=384, height=288
SupportedPreviewSizes: width=352, height=288
SupportedPreviewSizes: width=320, height=240
SupportedPreviewSizes: width=240, height=320
SupportedPreviewSizes: width=240, height=160
SupportedPreviewSizes: width=176, height=144
SupportedPreviewSizes: width=144, height=176
SupportedPreviewSizes: width=160, height=120
```
这里尺寸的比例一般都是4:3、16:9，其他比例是在此基础上裁剪出来的
## 选取预览尺寸
在相同宽高比下，选择最接近View的宽高，避免过大的预览尺寸, 造成性能损耗, 引起预览卡顿。   
在[cameraview](https://github.com/google/cameraview)源码中，默认定义的宽高比`AspectRatio DEFAULT_ASPECT_RATIO = AspectRatio.of(4, 3)`
### Camera1
``` java
private Size chooseOptimalSize(SortedSet<Size> sizes) {
    if (!mPreview.isReady()) { // Not yet laid out
        return sizes.first(); // Return the smallest size
    }
    int desiredWidth;
    int desiredHeight;
    final int surfaceWidth = mPreview.getWidth();
    final int surfaceHeight = mPreview.getHeight();
    if (isLandscape(mDisplayOrientation)) {
        desiredWidth = surfaceHeight;
        desiredHeight = surfaceWidth;
    } else {
        desiredWidth = surfaceWidth;
        desiredHeight = surfaceHeight;
    }
    Size result = null;
    for (Size size : sizes) { // Iterate from small to large
        if (desiredWidth <= size.getWidth() && desiredHeight <= size.getHeight()) {
            return size;

        }
        result = size;
    }
    return result;
}
```
区分了横竖屏，然后得到尺寸中宽和高等于或者大于View的宽高的尺寸。
### Camera2
``` java
private Size chooseOptimalSize() {
    int surfaceLonger, surfaceShorter;
    final int surfaceWidth = mPreview.getWidth();
    final int surfaceHeight = mPreview.getHeight();
    if (surfaceWidth < surfaceHeight) {
        surfaceLonger = surfaceHeight;
        surfaceShorter = surfaceWidth;
    } else {
        surfaceLonger = surfaceWidth;
        surfaceShorter = surfaceHeight;
    }
    SortedSet<Size> candidates = mPreviewSizes.sizes(mAspectRatio);

    // Pick the smallest of those big enough
    for (Size size : candidates) {
        if (size.getWidth() >= surfaceLonger && size.getHeight() >= surfaceShorter) {
            return size;
        }
    }
    // If no size is big enough, pick the largest one.
    return candidates.last();
}
```
先判断View宽高，区分其中较大值和较小值，然后再得到尺寸中宽和高大于或者等于View的较大值和较小值的尺寸。

## 拍照尺寸
代码也是从[cameraview](https://github.com/google/cameraview)中截取出来的
### Camera1
``` java
mPictureSizes.clear();
for (Camera.Size size : mCameraParameters.getSupportedPictureSizes()) {
    Log.d("DEBUG", "###### SupportedPictureSizes: width=" + size.width + ", height="
            + size.height);
    mPictureSizes.add(new Size(size.width, size.height));
}
```
### Camera2
``` java
protected void collectPictureSizes(SizeMap sizes, StreamConfigurationMap map) {
    for (android.util.Size size : map.getOutputSizes(ImageFormat.JPEG)) {
        mPictureSizes.add(new Size(size.getWidth(), size.getHeight()));
    }
}
```
在红米Note 5A手机支持的拍照尺寸：
``` java
SupportedPictureSizes: width=4160, height=3120
SupportedPictureSizes: width=4160, height=2340
SupportedPictureSizes: width=4096, height=3072
SupportedPictureSizes: width=4096, height=2304
SupportedPictureSizes: width=4000, height=3000
SupportedPictureSizes: width=3840, height=2160
SupportedPictureSizes: width=3264, height=2448
SupportedPictureSizes: width=3200, height=2400
SupportedPictureSizes: width=2976, height=2976
SupportedPictureSizes: width=2592, height=1944
SupportedPictureSizes: width=2592, height=1458
SupportedPictureSizes: width=2688, height=1512
SupportedPictureSizes: width=2304, height=1728
SupportedPictureSizes: width=2048, height=1536
SupportedPictureSizes: width=2336, height=1314
SupportedPictureSizes: width=1920, height=1080
SupportedPictureSizes: width=1600, height=1200
SupportedPictureSizes: width=1440, height=1080
SupportedPictureSizes: width=1280, height=960
SupportedPictureSizes: width=1280, height=768
SupportedPictureSizes: width=1280, height=720
SupportedPictureSizes: width=1200, height=1200
SupportedPictureSizes: width=1024, height=768
SupportedPictureSizes: width=800, height=600
SupportedPictureSizes: width=864, height=480
SupportedPictureSizes: width=800, height=480
SupportedPictureSizes: width=720, height=480
SupportedPictureSizes: width=640, height=480
SupportedPictureSizes: width=640, height=360
SupportedPictureSizes: width=480, height=640
SupportedPictureSizes: width=480, height=360
SupportedPictureSizes: width=480, height=320
SupportedPictureSizes: width=352, height=288
SupportedPictureSizes: width=320, height=240
SupportedPictureSizes: width=240, height=320
```
这里尺寸的比例一般也是4:3、16:9
## 选取拍照尺寸
Camaer1和Camera2都是一样的逻辑，选取固定宽高比例中的最大尺寸，这样拍摄的图片最清晰。
``` java
Size largest = mPictureSizes.sizes(mAspectRatio).last();
```

# 方向
这里的设置方向有两种：图像预览方向和拍照方向。在这之前，需要先介绍几个概念：
* 屏幕坐标方向
* 设备自然方向
* 摄像头传感器方向
* 相机预览方向

## 屏幕坐标方向
在Android系统中，以屏幕左上角为坐标系统的原点(0,0)坐标，该坐标系是固定不变的，不会因为设备方向的变化而改变。
![屏幕坐标方向](https://s2.ax1x.com/2020/01/31/11sOYQ.png)
## 屏幕自然方向
每个设备都有一个自然方向，手机和平板自然方向不一样，如图所示，这里盗个图：
[![设备自然方向](https://s2.ax1x.com/2020/01/31/11ySO0.md.png)](https://imgchr.com/i/11ySO0)
默认情况下，平板的自然方向是横屏，而手机的自然方向是竖屏方向。Android系统可以通过View的`OrientationEventListener`监听设备方向，回调方法：
``` java
abstract public void onOrientationChanged(int orientation);
```
`onOrientationChanged`返回0到359的角度，其中0表示自然方向。

## 摄像头传感器方向
[![摄像头传感器方向](https://s2.ax1x.com/2020/01/31/11sXWj.md.png)](https://imgchr.com/i/11sXWj)
手机相机的图像数据都是来自于摄像头硬件的图像传感器，这个传感器在被固定到手机上后有一个默认的取景方向，方向一般是和手机横屏方向一致，如上图所示。  

## 相机预览方向
将摄像头传感器捕获的图像，显示在屏幕上的方向，就是相机预览方向。默认情况下，和摄像头传感器方向一致，可以通过Camera API进行改变。    
Camaer1可以使用`setDisplayOrientation`设置预览方向，Camera2则可以通过TextureView来实现。  
不同的摄像头位置，`orientation`是不一样的，orientation就是摄像头传感器方向顺时针旋转到屏幕自然方向的角度。
### 后置
后置的`orientation`90
[![后置](https://s2.ax1x.com/2020/02/01/18NuGt.md.jpg)](https://imgchr.com/i/18NuGt)
对横屏来说，屏幕的自然方向和相机的摄像头传感器方向一致的。   
对竖屏来说，看到的图像逆时针旋转了90度，因此预览方向需要顺时针旋转90度，才能与屏幕的自然方向保持一致。
### 前置
前置的`orientation`270，收集到图像后(没有经过镜像处理)，但是要显示到屏幕上，就要按照屏幕自然方向的坐标系来进行显示，需要顺时针旋转270度，才能和设备自然方向一致。预览的时候，做了镜像处理，所以只需要顺时针旋转90度，就能和设置自然方向一致。   
那么Camera1和Camera2具体设置预览方向的代码，来自[cameraview](https://github.com/google/cameraview)：   

**Camera1**
``` java
private int calcDisplayOrientation(int screenOrientationDegrees) {
    if (mCameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        return (360 - (mCameraInfo.orientation + screenOrientationDegrees) % 360) % 360;    // compensate the mirror
    } else {  // back-facing
        return (mCameraInfo.orientation - screenOrientationDegrees + 360) % 360;
    }
}
```
代码中区分了前置和后置摄像头。   
* 后置：`(mCameraInfo.orientation - screenOrientationDegrees + 360) % 360`，恢复到自然方向需要顺时针旋转，而屏幕逆时针旋转正好抵掉了摄像头的旋转，两者差值+360取模。
* 前置：`(mCameraInfo.orientation + screenOrientationDegrees) % 360`，屏幕竖直方向看到的是一个镜像，360-`(mCameraInfo.orientation + screenOrientationDegrees) % 360`，顺时针旋转这个差值可以到自然方向，只不过这是个镜像，左右翻转了    

**Camera2**
使用的TextureView的setTransform进行旋转，并有区分横竖屏。
``` java
/**
* Configures the transform matrix for TextureView based on {@link #mDisplayOrientation} and
* the surface size.
*/
void configureTransform() {
    Matrix matrix = new Matrix();
    if (mDisplayOrientation % 180 == 90) {
        final int width = getWidth();
        final int height = getHeight();
        // Rotate the camera preview when the screen is landscape.
        matrix.setPolyToPoly(
                new float[]{
                        0.f, 0.f, // top left
                        width, 0.f, // top right
                        0.f, height, // bottom left
                        width, height, // bottom right
                }, 0,
                mDisplayOrientation == 90 ?
                        // Clockwise
                        new float[]{
                                0.f, height, // top left
                                0.f, 0.f, // top right
                                width, height, // bottom left
                                width, 0.f, // bottom right
                        } : // mDisplayOrientation == 270
                        // Counter-clockwise
                        new float[]{
                                width, 0.f, // top left
                                width, height, // top right
                                0.f, 0.f, // bottom left
                                0.f, height, // bottom right
                        }, 0,
                4);
    } else if (mDisplayOrientation == 180) {
        matrix.postRotate(180, getWidth() / 2, getHeight() / 2);
    }
    mTextureView.setTransform(matrix);
}
```
## 拍照方向
设置预览方向并不会改变拍出照片的方向。   
对于后置相机，相机采集到的图像和相机预览的图像是一样的，只需要旋转后置相机orientation度。
对于前置相机来说，相机预览的图像和相机采集到的图像是镜像关系。  
采集的图像：顺时针旋转270度后，与屏幕自然方向一致。   
预览的图像：顺时针旋转90度后，与屏幕自然方向一致。     
最后盗用一张图来说明：
![拍照方向](https://s2.ax1x.com/2020/02/01/18NlM8.jpg)
### Camera1
使用`mCameraParameters.setRotation()`设置拍照后图像方向：
``` java
mCameraParameters.setRotation(calcCameraRotation(displayOrientation));
......
/**
* Calculate camera rotation
*
* This calculation is applied to the output JPEG either via Exif Orientation tag
* or by actually transforming the bitmap. (Determined by vendor camera API implementation)
*
* Note: This is not the same calculation as the display orientation
*
* @param screenOrientationDegrees Screen orientation in degrees
* @return Number of degrees to rotate image in order for it to view correctly.
*/
private int calcCameraRotation(int screenOrientationDegrees) {
    if (mCameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        return (mCameraInfo.orientation + screenOrientationDegrees) % 360;
    } else {  // back-facing
        final int landscapeFlip = isLandscape(screenOrientationDegrees) ? 180 : 0;
        return (mCameraInfo.orientation + screenOrientationDegrees + landscapeFlip) % 360;
    }
}
```
相机采集到的图像，只需要旋转相机orientation度。
### Camera2
根据`CameraCharacteristics.SENSOR_ORIENTATION`，使用`captureRequest`设置了JPEG图像的旋转方向。
``` java
// Calculate JPEG orientation.
@SuppressWarnings("ConstantConditions")
int sensorOrientation = mCameraCharacteristics.get(
        CameraCharacteristics.SENSOR_ORIENTATION);
captureRequestBuilder.set(CaptureRequest.JPEG_ORIENTATION,
        (sensorOrientation +
                mDisplayOrientation * (mFacing == Constants.FACING_FRONT ? 1 : -1) +
                360) % 360);
```

# 图像数据
Android Camera默认返回的数据格式是NV21。Camera1通过`mParameters.setPreviewFormat()`设置，Camera2通过`ImageReader.newInstance()`设置。  
ImageFormat枚举了很多种图片格式，其中ImageFormat.NV21和ImageFormat.YV12是官方推荐的格式，NV21、YV12格式都属于 YUV 格式，也可以表示为YCbCr，Cb、Cr的含义等同于U、V。   
### YUV
YUV是一种颜色编码方法，和它类似的还有RGB颜色编码方法，主要应用于电视系统和模拟视频领域。其中YUV代表三个分量，Y 代表明亮度，U 和 V 表示的是色度，色度又定义了颜色的两个方面：色调和饱和度。将Y与UV分离，没有UV信息一样可以显示完整的图像，但是只能显示灰度图。   
### YUV采样格式
YUV 图像的主流采样方式有如下三种：
* YUV 4:4:4 采样：每一个Y对应一组UV分量
* YUV 4:2:2 采样：每两个Y共用一组UV分量
* YUV 4:2:0 采样：每四个Y共用一组UV分量

盗个图说明比较清晰，黑点表示采样该像素点的Y分量，空心圆圈表示采用该像素点的UV分量
[![YUV](https://s2.ax1x.com/2020/02/01/1G8a5t.md.jpg)](https://imgchr.com/i/1G8a5t)
### YUV存储格式
有两种存储格式，planar和packed。
* planar：先连续存储所有像素点的Y，紧接着存储所有像素点的U，随后是所有像素点的V
* packed：每个像素点的Y,U,V是连续交替存储

YUV格式信息可以参考：[YUV pixel formats](https://www.fourcc.org/yuv.php)   
根据采样方式和存储格式的不同，形成了多种YUV格式，常见的YUV格式：

| 采样/格式  |  |    | |
|:---------:|------------|--------------|--------------|
| YUV422      | YUVY 格式    |  UYVY 格式 | YUV422P 格式|
| YUV420     |YUV420P<br>(YV12、YU12格式)|YUV420P<br>(NV12、NV21格式)  ||

#### YUVY格式
YUVY格式属于packed存储格式，相邻的两个Y共用其相邻的两个U、V
``` java
Y0 UO Y1 V0 Y2 U2 Y3 V2
```
Y0、Y1共用 U0、V0   
Y2、Y3共用 U2、V2
#### UYVY格式
UYVY格式也属于packed存储格式，与YUYV格式不同的是UV的排列顺序不一样而已
#### YUV422P格式
YUV422P格式属于planar存储格式，先连续存储所有像素点的Y，紧接着存储所有像素点的U，随后是所有像素点的V
#### YV12、YU12格式
YU12和YV12格式都属于YUV420P格式，YUV420P是planar存储格式。先存储所有Y，然后在存储U、V。     
YU12和YV12的区别在于YU12是先Y再U后V，而YV12是先Y再V后U。
#### NV12、NV21格式
NV12、NV21格式YUV420SP格式，YUV420SP也是planar存储格式。先存储所有Y，然后按照UV或者VU的交替顺序进行存储。   
NV12格式先存储Y，然后UV再进行交替存储。   
NV21格式则是先存储Y，然后VU再进行交替存储。  
最后盗用一个数据格式的总结：   
``` xml
YV21: YYYYYYYY UU VV => YUV420P   
YV12: YYYYYYYY VV UU => YUV420P   
NV12: YYYYYYYY UV UV => YUV420SP   
NV21: YYYYYYYY VU VU => YUV420SP
```

Android Camera 默认数据格式是 NV21，Camera1直接设置`mParameters.setPreviewFormat(ImageFormat.NV21)`，然后拍照回调中的 raw data 数据返回就是 NV21的。   
Camera2通过`ImageReader.newInstance()`设置，但是不能直接设置格式`ImageFormat.NV21`，在源码中有段代码：
``` java
if (format == ImageFormat.NV21) {
    throw new IllegalArgumentException(
            "NV21 format is not supported");
}
```
在最新的`ImageFormat.NV21`上有说明：
```  java
YCrCb format used for images, which uses the NV21 encoding format.   
This is the default format for android.hardware.Camera preview images,
when not otherwise set with android.hardware.Camera.Parameters.setPreviewFormat(int).
For the android.hardware.camera2 API, the YUV_420_888 format is recommended for YUV output instead.
```
Camera2建议使用`YUV_420_888`来替代，所以要得到NV21的数据需要进行数据转化，具体可以参考[Image类浅析(结合YUV_420_888)](https://www.polarxiong.com/archives/Android-Image%E7%B1%BB%E6%B5%85%E6%9E%90-%E7%BB%93%E5%90%88YUV_420_888.html)
# 参考
* [Android相机开发和遇到的坑](https://blog.csdn.net/xx326664162/article/details/53350551)
* [Android Camera2 教程 · 第三章 · 预览](https://www.jianshu.com/p/067889611ae7)
* [Android 相机开发中的尺寸和方向问题](https://glumes.com/post/android/android-camera-aspect-ratio-and-orientation/)
* [【腾讯优测干货分享】Android 相机预览方向及其适配探索](https://blog.csdn.net/tencent_bugly/article/details/53375311)
* [一文读懂 YUV 的采样与格式](https://juejin.im/post/5ce497285188252dd500c304)
* [图文详解YUV420数据格式](https://www.cnblogs.com/azraelly/archive/2013/01/01/2841269.html)
* [Image类浅析(结合YUV_420_888)](https://www.polarxiong.com/archives/Android-Image%E7%B1%BB%E6%B5%85%E6%9E%90-%E7%BB%93%E5%90%88YUV_420_888.html)
