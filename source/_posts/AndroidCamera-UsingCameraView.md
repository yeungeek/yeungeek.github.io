---
title: Android Camera-CameraView和CameraX使用
date: 2020-01-21 16:21:38
tags:
   - Camera
   - Camera2
   - SurfaceView
   - TextureView
   - CameraX
   - CameraView
categories:
   - Camera 
---
前面两篇介绍了Camera1和Camera2的使用，发现Camera API从1到2的变化非常大，Camera2的复杂度提升了不少，官方为了让我们更容易使用Camera，出了个一个官方的库[cameraview](https://github.com/google/cameraview)。不过这个库已经Deprecated，官方建议使用[Jetpack CameraX](https://developer.android.com/jetpack/androidx/releases/camerax) 替代。本篇文章就介绍下CameraView和CameraX的使用
<!--more-->
# CameraView
CameraView的目的就是帮助开发者能够快速集成Camera1和Camera2的特性，可以用下面这张表来说明：

| API Level | Camera API | Preview View |
|:---------:|------------|--------------|
| 9-13      | Camera1    | SurfaceView  |
| 14-20     | Camera1    | TextureView  |
| 21-23     | Camera2    | TextureView  |
| 24        | Camera2    | SurfaceView  |

## 开发流程
### CameraView定义

xml中定义
``` xml
<com.google.android.cameraview.CameraView
    android:id="@+id/camera"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:keepScreenOn="true"
    android:adjustViewBounds="true"
    app:autoFocus="true"
    app:aspectRatio="4:3"
    app:facing="back"
    app:flash="auto"/>
```
xml中可以配置：
* autoFocus：是否自动对焦
* aspectRatio：预览画面比例
* facing：前后摄像头
* flash：闪光灯模式

### 增加生命周期
``` java
@Override
protected void onResume() {
    super.onResume();
    mCameraView.start();
}

@Override
protected void onPause() {
    mCameraView.stop();
    super.onPause();
}
```
这样声明后，就可以完成预览的工作了

### 相机状态回调
在xml声明CameraView后，增加回调
``` java
if (mCameraView != null) {
    mCameraView.addCallback(mCallback);
}
...
private CameraView.Callback mCallback
            = new CameraView.Callback() {

    @Override
    public void onCameraOpened(CameraView cameraView) {
        Log.d(TAG, "onCameraOpened");
    }

    @Override
    public void onCameraClosed(CameraView cameraView) {
        Log.d(TAG, "onCameraClosed");
    }

    @Override
    public void onPictureTaken(CameraView cameraView, final byte[] data) {
        Log.d(TAG, "onPictureTaken " + data.length);
        Toast.makeText(cameraView.getContext(), R.string.picture_taken, Toast.LENGTH_SHORT)
                .show();
        getBackgroundHandler().post(new Runnable() {
            @Override
            public void run() {
                File file = new File(getExternalFilesDir(Environment.DIRECTORY_PICTURES),
                        "picture.jpg");
                Log.d(TAG, "onPictureTaken file path: " + file.getPath());
                OutputStream os = null;
                try {
                    os = new FileOutputStream(file);
                    os.write(data);
                    os.close();
                } catch (IOException e) {
                    Log.w(TAG, "Cannot write to " + file, e);
                } finally {
                    if (os != null) {
                        try {
                            os.close();
                        } catch (IOException e) {
                            // Ignore
                        }
                    }
                }
            }
        });
    }

};
```
有三个回调方法，相机打开，相机关闭，和拍照。
### 拍照
``` java
mCameraView.takePicture();
```
就是这么简单，点击后拍照，然后回调中处理图像数据

# CameraX
CameraX 是一个 Jetpack 支持库，目的是简化Camera的开发工作，它是基于Camera2 API的基础，向后兼容至 Android 5.0（API 级别 21）。   
它有以下几个特性：
* 易用性，只需要几行代码就可以实现预览和拍照
* 保持设备的一致性，在不同相机设备上，对宽高比、屏幕方向、旋转、预览大小和高分辨率图片大小，做到都可以正常使用
* 相机特性的扩展，增加人像、HDR、夜间模式和美颜等功能

## 开发流程
### 库引用
目前CameraX最新版本是`1.0.0-alpha06`，在app的build.gradle引用：
``` gradle
dependencies {
    // CameraX core library.
    def camerax_version = "1.0.0-alpha06"
    implementation "androidx.camera:camera-core:${camerax_version}"
    // If you want to use Camera2 extensions.
    implementation "androidx.camera:camera-camera2:${camerax_version}"

    def camerax_view_version = "1.0.0-alpha03"
    def camerax_ext_version = "1.0.0-alpha03"
    //other
    // If you to use the Camera View class
    implementation "androidx.camera:camera-view:$camerax_view_version"
    // If you to use Camera Extensions
    implementation "androidx.camera:camera-extensions:$camerax_ext_version"
}
```
因为CameraX是一个 Jetpack 支持库，相机的打开和释放都是使用了Jetpack的Lifecycle来进行处理。
### 预览
预览参数设置，使用PreviewConfig.Builder()实现：
``` java
PreviewConfig config = new PreviewConfig.Builder()
                .setLensFacing(CameraX.LensFacing.BACK)
                .setTargetRotation(mTextureView.getDisplay().getRotation())
                .setTargetResolution(new Size(640, 480))
                .build();

Preview preview = new Preview(config);
preview.setOnPreviewOutputUpdateListener(new Preview.OnPreviewOutputUpdateListener() {
    @Override
    public void onUpdated(@NonNull Preview.PreviewOutput output) {
        if (mTextureView.getParent() instanceof ViewGroup) {
            ViewGroup viewGroup = (ViewGroup) mTextureView.getParent();
            viewGroup.removeView(mTextureView);
            viewGroup.addView(mTextureView, 0);

            mTextureView.setSurfaceTexture(output.getSurfaceTexture());
            updateTransform();
        }
    }
});

//lifecycle
CameraX.bindToLifecycle(this, preview);
```
PreivewConfig.Builder可以设置的属性很多，这里只设置了摄像头、旋转方向、预览分辨率，还有很多其他方法，大家可以自行试验。  
在preview回调监听中，把output的SurfaceTexture设置到mTextureView中，实现图像预览，最后增加Lifecycle的绑定。
### 拍照
``` java
ImageCaptureConfig captureConfig = new ImageCaptureConfig.Builder()
        .setTargetAspectRatio(AspectRatio.RATIO_16_9)
        .setCaptureMode(ImageCapture.CaptureMode.MIN_LATENCY)
        .setTargetRotation(getWindowManager().getDefaultDisplay().getRotation())
        .build();

ImageCapture imageCapture = new ImageCapture(captureConfig);
mTakePicture.setOnClickListener((view) -> {
    final File file = new File(getExternalMediaDirs()[0], System.currentTimeMillis() + ".jpg");
    Log.d("DEBUG", "##### file path: " + file.getPath());
    imageCapture.takePicture(file, ContextCompat.getMainExecutor(getApplicationContext()), new ImageCapture.OnImageSavedListener() {
        @Override
        public void onImageSaved(@NonNull File file) {
            Log.d("DEBUG", "##### onImageSaved: " + file.getPath());
        }

        @Override
        public void onError(@NonNull ImageCapture.ImageCaptureError imageCaptureError, @NonNull String message, @Nullable Throwable cause) {
            Log.d("DEBUG", "##### onError: " + message);
        }
    });
});

CameraX.bindToLifecycle(this, preview, imageCapture);
```
拍照的参数通过`ImageCaptureConfig.Builder`设置，这里只设置了图片宽高比、拍摄模式和旋转方向，还有很多其他方法，大家可以自行试验。  
真正调用拍照的方法：
* takePicture(OnImageCapturedListener)：此方法为拍摄的图片提供内存缓冲区。
* takePicture(File, OnImageSavedListener)：此方法将拍摄的图片保存到提供的文件位置。
* takePicture(File, OnImageSavedListener, Metadata)：此方法可用于指定要嵌入已保存文件的 Exif 中的元数据。

例子调用的是takePicture(File, OnImageSavedListener)，直接存为文件。最后再增加Lifecycle的绑定。

### 图片分析
``` java
ImageAnalysisConfig analysisConfig = new ImageAnalysisConfig.Builder()
        .setImageReaderMode(ImageAnalysis.ImageReaderMode.ACQUIRE_LATEST_IMAGE)
        .build();

ImageAnalysis imageAnalysis = new ImageAnalysis(analysisConfig);
imageAnalysis.setAnalyzer(ContextCompat.getMainExecutor(getApplicationContext()),
        new LuminosityAnalyzer());

CameraX.bindToLifecycle(this, preview, imageCapture, imageAnalysis);

...
private class LuminosityAnalyzer implements ImageAnalysis.Analyzer {
    private long lastAnalyzedTimestamp = 0L;

    @Override
    public void analyze(ImageProxy image, int rotationDegrees) {
        final Image img = image.getImage();
        if (img != null) {
            Log.d("DEBUG", img.getWidth() + "," + img.getHeight());
        }
    }
}
```
图片分析，不是必要的步骤，但是ImageAnalysis，可以对每帧图像进行分析。  
设置参数通过`ImageAnalysisConfig.Builder()`，这里只设置了`ImageReaderMode`，它有两种模式：
* 阻止模式(ImageReaderMode.ACQUIRE_NEXT_IMAGE)：就是Camera2中的acquireNextImage()，获取下一个最新的可用Image
* 非阻止模式(ImageReaderMode.ACQUIRE_LATEST_IMAGE)：Camera2中的acquireLatestImage()，获得图像队列中最新的图片，并且会清空队列,删除已有的旧的图像

最后还是增加Lifecycle的绑定。CameraX的使用也非常简单，把Camera2中复杂的API封装到统一的config中，只需要几行代码，就实现需要的功能。    
文章中涉及到的[代码](https://github.com/yeungeek/AndroidRoad/blob/master/CameraSample/app/src/main/java/com/yeungeek/camerasample/camerax/CameraXActivity.java)

# 参考
* [CameraX](https://developer.android.com/training/camerax)
* [Android Camera 编程从入门到精通](https://www.jianshu.com/p/f63f296a920b)
* [Google Jetpack 新组件 CameraX 介绍与实践](https://glumes.com/post/android/google-jetpack-camerax/)
