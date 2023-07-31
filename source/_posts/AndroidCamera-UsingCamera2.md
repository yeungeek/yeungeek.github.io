---
title: Android Camera-Camera2使用
date: 2020-01-19 19:30:54
tags:
   - Camera
   - Camera2
   - SurfaceView
   - TextureView
categories:
   - Camera 
---
上篇文章介绍了Camera1的使用，本篇介绍Camera2的使用。  
Camera2(android.hardware.camera2)是从 Android 5.0 L 版本开始引入的，并且废弃了旧的相机框架Camera1(android.hardware.Camera)。
相比于Camera1，Camera2架构上也发生了变化，API上的使用难度也增加了。Camera2将相机设备模拟成一个管道，它按顺序处理每一帧的请求并返回请求结果给客户端。
<!--more-->
# 设计框架
来自官网的模型图，展示了相关的工作流程
![相机核心操作模型](https://s2.ax1x.com/2020/01/25/1eqOyj.png)
重新设计 Android Camera API 的目的在于大幅提高应用对于 Android 设备上的相机子系统的控制能力，同时重新组织 API，提高其效率和可维护性。  
在CaptureRequest中设置不同的Surface用于接收不同的图片数据，最后从不同的Surface中获取到图片数据和包含拍照相关信息的CaptureResult。  
## 优点
通过设计框架的改造和优化，Camera2具备了以下优点:
* 改进了新硬件的性能。Supported Hardware Level的概念，不同厂商对Camera2的支持程度不同，从低到高有LEGACY、LIMITED、FULL 和 LEVEL_3四个级别
* 以更快的间隔拍摄图像
* 显示来自多个摄像机的预览
* 直接应用效果和滤镜

# 开发流程
框架上的变化，对整个使用流程变化也非常大，首先了解一些主要的开发类
## 类
### CameraManager
相机系统服务，用于管理和连接相机设备
### CameraDevice
相机设备类，和Camera1中的Camera同级
### CameraCharacteristics
主要用于获取相机信息，内部携带大量的相机信息，包含摄像头的正反(`LENS_FACING`)、AE模式、AF模式等，和Camera1中的Camera.Parameters类似
### CaptureRequest
相机捕获图像的设置请求，包含传感器，镜头，闪光灯等
### CaptureRequest.Builder
CaptureRequest的构造器，使用Builder模式，设置更加方便
### CameraCaptureSession
请求抓取相机图像帧的会话，会话的建立主要会建立起一个通道。一个CameraDevice一次只能开启一个CameraCaptureSession。
源端是相机，另一端是 Target，Target可以是Preview，也可以是ImageReader。
### ImageReader
用于从相机打开的通道中读取需要的格式的原始图像数据，可以设置多个ImageReader。

## 流程
![Camera2开发流程](https://s2.ax1x.com/2020/01/29/1QX6XV.png)
### 获取CameraManager
``` java
CameraManager cameraManager = (CameraManager) getSystemService(Context.CAMERA_SERVICE);
```

### 获取相机信息
``` java
for (String cameraId : cameraManager.getCameraIdList()) {
    CameraCharacteristics characteristics = cameraManager.getCameraCharacteristics(cameraId);

    Integer facing = characteristics.get(CameraCharacteristics.LENS_FACING);
    if (null != facing && facing == CameraCharacteristics.LENS_FACING_FRONT) {
        continue;
    }
    ....
}
```
这里默认选择前置摄像头，并获取相关相机信息。
### 初始化ImageReader
``` java
mImageReader = ImageReader.newInstance(largest.getWidth(), largest.getHeight(), ImageFormat.JPEG, 2);
mImageReader.setOnImageAvailableListener(new ImageReader.OnImageAvailableListener() {
    @Override
    public void onImageAvailable(ImageReader reader) {
        Log.d("DEBUG", "##### onImageAvailable: " + mFile.getPath());
        mBackgroundHandler.post(new ImageSaver(reader.acquireNextImage(), mFile));
    }
}, mBackgroundHandler);
```
`ImageReader`是获取图像数据的重要途径，通过它可以获取到不同格式的图像数据，例如JPEG、YUV、RAW等。通过`ImageReader.newInstance(int width, int height, int format, int maxImages)`创建`ImageReader`对象，有4个参数：
* width：图像数据的宽度
* height：图像数据的高度
* format：图像数据的格式，例如`ImageFormat.JPEG`，`ImageFormat.YUV_420_888`等
* maxImages：最大Image个数，Image对象池的大小，指定了能从ImageReader获取Image对象的最大值，过多获取缓冲区可能导致OOM，所以最好按照最少的需要去设置这个值

ImageReader其他相关的方法和回调：
* ImageReader.OnImageAvailableListener：有新图像数据的回调
* acquireLatestImage()：从ImageReader的队列里面，获取最新的Image，删除旧的，如果没有可用的Image，返回null
* acquireNextImage()：获取下一个最新的可用Image，没有则返回null
* close()：释放与此ImageReader关联的所有资源
* getSurface()：获取为当前ImageReader生成Image的Surface

### 打开相机设备
``` java
try {
    if (!mCameraOpenCloseLock.tryAcquire(2500, TimeUnit.MILLISECONDS)) {
        throw new RuntimeException("Time out waiting to lock camera opening.");
    }

    cameraManager.openCamera(mCameraId, mStateCallback, mBackgroundHandler);
} catch (Exception e) {
    e.printStackTrace();
}
```
`cameraManager.openCamera(@NonNull String cameraId,@NonNull final CameraDevice.StateCallback callback, @Nullable Handler handler)`的三个参数:
* cameraId：摄像头的唯一标识
* callback：设备连接状态变化的回调
* handler：回调执行的Handler对象，传入null则使用当前的主线程Handler

其中callback回调：
``` java
private final CameraDevice.StateCallback mStateCallback = new CameraDevice.StateCallback() {
    @Override
    public void onOpened(@NonNull CameraDevice camera) {
        mCameraOpenCloseLock.release();
        mCameraDevice = camera;
        createCameraPreviewSession();
    }

    @Override
    public void onDisconnected(@NonNull CameraDevice camera) {
        mCameraOpenCloseLock.release();
        camera.close();
        mCameraDevice = null;
    }

    @Override
    public void onError(@NonNull CameraDevice camera, int error) {
        mCameraOpenCloseLock.release();
        camera.close();
        mCameraDevice = null;
    }

    @Override
    public void onClosed(@NonNull CameraDevice camera) {
        super.onClosed(camera);
    }
};
```
* onOpened：表示相机打开成功，可以真正开始使用相机，创建Capture会话
* onDisconnected：当相机断开连接时回调该方法，需要进行释放相机的操作
* onError：当相机打开失败时，需要进行释放相机的操作
* onClosed：调用Camera.close()后的回调方法

### 创建Capture会话
在CameraDevice.StateCallback的onOpened回调中执行：
``` java
private void createCameraPreviewSession() {
    SurfaceTexture texture = mTextureView.getSurfaceTexture();
    assert texture != null;
    texture.setDefaultBufferSize(mPreviewSize.getWidth(), mPreviewSize.getHeight());
    Surface surface = new Surface(texture);

    try {
        mPreviewRequestBuilder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
        mPreviewRequestBuilder.addTarget(surface);

        // Here, we create a CameraCaptureSession for camera preview.
        mCameraDevice.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
                new CameraCaptureSession.StateCallback() {

                    @Override
                    public void onConfigured(@NonNull CameraCaptureSession cameraCaptureSession) {
                        // The camera is already closed
                        if (null == mCameraDevice) {
                            return;
                        }

                        // When the session is ready, we start displaying the preview.
                        mCaptureSession = cameraCaptureSession;
                        try {
                            // Auto focus should be continuous for camera preview.
                            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
                                    CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
                            // Flash is automatically enabled when necessary.
                            setAutoFlash(mPreviewRequestBuilder);

                            // Finally, we start displaying the camera preview.
                            mPreviewRequest = mPreviewRequestBuilder.build();
                            mCaptureSession.setRepeatingRequest(mPreviewRequest,
                                    mCaptureCallback, mBackgroundHandler);
                        } catch (CameraAccessException e) {
                            e.printStackTrace();
                        }
                    }

                    @Override
                    public void onConfigureFailed(
                            @NonNull CameraCaptureSession cameraCaptureSession) {
                        Toast.makeText(Camera2Activity.this, "configureFailed", Toast.LENGTH_SHORT).show();
                    }
                }, null
        );
    } catch (CameraAccessException e) {
        e.printStackTrace();
    }
}
```
这段的代码核心方法是`mCameraDevice.createCaptureSession()`创建Capture会话，它接受了三个参数：
* outputs：用于接受图像数据的surface集合，这里传入的是一个preview的surface
* callback：用于监听 Session 状态的CameraCaptureSession.StateCallback对象
* handler：用于执行CameraCaptureSession.StateCallback的Handler对象，传入null则使用当前的主线程Handler

### 创建CaptureRequest
CaptureRequest是向CameraCaptureSession提交Capture请求时的信息载体，其内部包括了本次Capture的参数配置和接收图像数据的Surface。   
``` Java
mPreviewRequestBuilder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
mPreviewRequestBuilder.addTarget(surface);
```
通过`CameraDevice.createCaptureRequest()`创建`CaptureRequest.Builder`对象，传入一个templateType参数，templateType用于指定使用何种模板创建`CaptureRequest.Builder`对象，templateType的取值：
* TEMPLATE_PREVIEW：预览模式
* TEMPLATE_STILL_CAPTURE：拍照模式
* TEMPLATE_RECORD：视频录制模式
* TEMPLATE_VIDEO_SNAPSHOT：视频截图模式
* TEMPLATE_MANUAL：手动配置参数模式

除了模式的配置，CaptureRequest还可以配置很多其他信息，例如图像格式、图像分辨率、传感器控制、闪光灯控制、3A(自动对焦-AF、自动曝光-AE和自动白平衡-AWB)控制等。在createCaptureSession的回调中可以进行设置
``` c++
// Auto focus should be continuous for camera preview.
mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
        CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
// Flash is automatically enabled when necessary.
setAutoFlash(mPreviewRequestBuilder);

// Finally, we start displaying the camera preview.
mPreviewRequest = mPreviewRequestBuilder.build();
```
代码中设置了AF为设置未图片模式下的连续对焦，并设置自动闪光灯。最后通过`build()`方法生成CaptureRequest对象。
### 预览
Camera2中，通过连续重复的Capture实现预览功能，每次Capture会把预览画面显示到对应的Surface上。连续重复的Capture操作通过`mCaptureSession.setRepeatingRequest(mPreviewRequest,mCaptureCallback, mBackgroundHandler)`实现，该方法有三个参数：
* request：CaptureRequest对象
* listener：监听Capture 状态的回调
* handler：用于执行CameraCaptureSession.CaptureCallback的Handler对象，传入null则使用当前的主线程Handler

停止预览使用`mCaptureSession.stopRepeating()`方法。
### 拍照
设置上面的request，session后，就可以真正的开始拍照操作
``` java
mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, mBackgroundHandler);
```
该方法也有三个参数，和mCaptureSession.setRepeatingRequest一样：
* request：CaptureRequest对象
* listener：监听Capture 状态的回调
* handler：用于执行CameraCaptureSession.CaptureCallback的Handler对象，传入null则使用当前的主线程Handler

这里设置了mCaptureCallback：
``` java
private CameraCaptureSession.CaptureCallback mCaptureCallback = new CameraCaptureSession.CaptureCallback() {
    @Override
    public void onCaptureProgressed(@NonNull CameraCaptureSession session, @NonNull CaptureRequest request, @NonNull CaptureResult partialResult) {
        process(partialResult);
    }

    @Override
    public void onCaptureCompleted(@NonNull CameraCaptureSession session, @NonNull CaptureRequest request, @NonNull TotalCaptureResult result) {
        process(result);
    }

    private void process(CaptureResult result) {
        switch (mState) {
            case STATE_PREVIEW: {
                // We have nothing to do when the camera preview is working normally.
                break;
            }
            case STATE_WAITING_LOCK: {
                Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
                Log.d("DEBUG", "##### process STATE_WAITING_LOCK: " + afState);
                if (afState == null) {
                    captureStillPicture();
                } else if (CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                        CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                    // CONTROL_AE_STATE can be null on some devices
                    Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                    if (aeState == null ||
                            aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                        mState = STATE_PICTURE_TAKEN;
                        captureStillPicture();
                    } else {
                        runPrecaptureSequence();
                    }
                }
                break;
            }
            case STATE_WAITING_PRECAPTURE: {
                // CONTROL_AE_STATE can be null on some devices
                Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                if (aeState == null ||
                        aeState == CaptureResult.CONTROL_AE_STATE_PRECAPTURE ||
                        aeState == CaptureRequest.CONTROL_AE_STATE_FLASH_REQUIRED) {
                    mState = STATE_WAITING_NON_PRECAPTURE;
                }
                break;
            }
            case STATE_WAITING_NON_PRECAPTURE: {
                // CONTROL_AE_STATE can be null on some devices
                Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                if (aeState == null || aeState != CaptureResult.CONTROL_AE_STATE_PRECAPTURE) {
                    mState = STATE_PICTURE_TAKEN;
                    captureStillPicture();
                }
                break;
            }
        }
    }
};
```
通过设置`mState`来区分当前状态，是在预览还是拍照

### 关闭相机
退到后台或者当前页面被关闭的时候，已经不需要使用相机了，需要进行相机关闭操作，释放资源，
``` java
private void closeCamera() {
    try {
        mCameraOpenCloseLock.acquire();
        if (null != mCaptureSession) {
            mCaptureSession.close();
            mCaptureSession = null;
        }
        if (null != mCameraDevice) {
            mCameraDevice.close();
            mCameraDevice = null;
        }
        if (null != mImageReader) {
            mImageReader.close();
            mImageReader = null;
        }
    } catch (InterruptedException e) {
        throw new RuntimeException("Interrupted while trying to lock camera closing.", e);
    } finally {
        mCameraOpenCloseLock.release();
    }
}
```
先后对CaptureSession，CameraDevice，ImageReader进行close操作，释放资源。
这里仅仅对Camera2基本使用流程做了介绍，一些更高级的用法需要大家自行去实践。在Camera1中需要对画面进行方向矫正，而Camera2是否需要呢，关于相机Orientation相关的知识，通过后面的章节再进行介绍。

文章中涉及到的[代码](https://github.com/yeungeek/AndroidRoad/blob/master/CameraSample/app/src/main/java/com/yeungeek/camerasample/camera2/Camera2Activity.java)

# 参考：
* [Camera](https://source.android.com/devices/camera)
* [Detecting camera features with Camera2](https://medium.com/google-developers/detecting-camera-features-with-camera2-61675bb7d1bf)
* [Android Camera 编程从入门到精通](https://www.jianshu.com/p/f63f296a920b)
* [Android Camera2 教程 · 第一章 · 概览](https://www.jianshu.com/p/9a2e66916fcb)
