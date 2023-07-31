---
title: Android Camera-CameraView源码分析
date: 2020-01-29 15:54:31
tags:
   - Camera
   - Camera2
   - SurfaceView
   - TextureView
   - CameraView
categories:
   - Camera 
---
在前面文章中已经介绍了如何使用 [CameraView](https://github.com/google/cameraview)，这是 Google 官方提供的库，并有相关 demo，因为 Android 的碎片化太严重，官方也是考虑到这些，才提供了[CameraView](https://github.com/google/cameraview)供大家学习和参考。     
<!-- more -->
# 源码分析
## 源码结构
[![源码结构](https://s2.ax1x.com/2020/02/08/1WWkn0.png)](https://imgchr.com/i/1WWkn0)
根据官方的说明： 

| API Level | Camera API | Preview View |
|:---------:|------------|--------------|
| 9-13      | Camera1    | SurfaceView  |
| 14-20     | Camera1    | TextureView  |
| 21-23     | Camera2    | TextureView  |
| 24        | Camera2    | SurfaceView  |

具体的实现都在[CameraView类](https://github.com/google/cameraview/blob/master/library/src/main/java/com/google/android/cameraview/CameraView.java)中。
## 类图
源码中涉及到的主要类之间的关系，是从最新的源码中查看到(跟官网的表格不太一样，最新的源码sdk最低版本为14)：
![CameraView](https://s2.ax1x.com/2020/02/13/1OoPpD.png)
* Camera 区分：Android5.0(21)以下使用 Camera1，以上使用 Camera2
* Preview View：Android6.0(23)以上使用SurfaceView(SurfaceView在Android7.0上增加了新特性(平移、旋转等))，这里应该是 Android7.0以上(>23)使用SurfaceView，其他都使用TextureView，最新的源码sdk最低版本要求14。

## CameraView
CameraView使用非常简单，在 CameraView 使用那篇文章已经做过详细说明。 
### 初始化
``` java
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
直接看下`CameraView`的构造函数：
``` java
public CameraView(Context context, AttributeSet attrs, int defStyleAttr) {
   super(context, attrs, defStyleAttr);
   ...
   // Internal setup
   // 1.创建预览视图
   final PreviewImpl preview = createPreviewImpl(context);
   mCallbacks = new CallbackBridge();
   // 2.根据 Android SDK 版本选择不同的 Camera
   if (Build.VERSION.SDK_INT < 21) {
      mImpl = new Camera1(mCallbacks, preview);
   } else if (Build.VERSION.SDK_INT < 23) {
      mImpl = new Camera2(mCallbacks, preview, context);
   } else {
      mImpl = new Camera2Api23(mCallbacks, preview, context);
   }
   // Attributes
   // 3. 读取自定义 View 属性, 设置相机摄像头位置、预览画面比例、对焦方式、闪光灯
   TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CameraView, defStyleAttr,
            R.style.Widget_CameraView);
   mAdjustViewBounds = a.getBoolean(R.styleable.CameraView_android_adjustViewBounds, false);
   setFacing(a.getInt(R.styleable.CameraView_facing, FACING_BACK));
   String aspectRatio = a.getString(R.styleable.CameraView_aspectRatio);
   if (aspectRatio != null) {
      setAspectRatio(AspectRatio.parse(aspectRatio));
   } else {
      setAspectRatio(Constants.DEFAULT_ASPECT_RATIO);
   }
   setAutoFocus(a.getBoolean(R.styleable.CameraView_autoFocus, true));
   setFlash(a.getInt(R.styleable.CameraView_flash, Constants.FLASH_AUTO));
   a.recycle();
   // Display orientation detector
   // 4. 增加旋转方向监听，设置相机的旋转方向
   mDisplayOrientationDetector = new DisplayOrientationDetector(context) {
      @Override
      public void onDisplayOrientationChanged(int displayOrientation) {
            mImpl.setDisplayOrientation(displayOrientation);
      }
   };
}
```
`createPreviewImpl`实现：
``` java
private PreviewImpl createPreviewImpl(Context context) {
   PreviewImpl preview;
   if (Build.VERSION.SDK_INT >= 23) {
      preview = new SurfaceViewPreview(context, this);
   } else {
      preview = new TextureViewPreview(context, this);
   }
   return preview;
}
```
这里主要代码角度看下CameraView版本选择策略，在最开始的已经说明。  
Camera的选择：api <21的，使用Camera1，>=21使用 Camera2，这里完全没有问题。    
Camera Preview的选择，存在几个疑问，api >=23使用SurfaceView，不过api 24的时候使用SurfaceView才是官方推荐的。  
> Starting in platform version N, SurfaceView’s window position is updated synchronously with other View rendering. This means that translating and scaling a SurfaceView on screen will not cause rendering artifacts. Such artifacts may occur on previous versions of the platform when its window is positioned asynchronously.

这里应该 api >23使用SurfaceView，其他情况使用TextureView，因为在工程中 定义了`minSdkVersion = 14`，api 14-23使用TextureView，表格可以更新为：

| API Level | Camera API | Preview View |
|:---------:|------------|--------------|
| 14-20     | Camera1    | TextureView  |
| 21-23     | Camera2    | TextureView  |
| 24        | Camera2    | SurfaceView  |

### Preview定义
`PreviewImpl`封装了预览控件的操作方法，`SurfaceViewPreview`和`TextureViewPreview`分别对应`SurfaceView`和`TextureView`的`PreviewImpl`实现。  
#### SurfaceViewPreview
实现很简单，直接加载有`SurfaceView`控件的布局，并封装了`SurfaceHolder`操作
#### TextureViewPreview
直接加载有`TextureView`控件布局，并监听`TextureView.SurfaceTextureListener`。

### Camera定义
`CameraViewImpl`定义了相机的各种操作，`Camera1`、`Camera2`、`Camera2Api23`都是`CameraViewImpl`的具体实现。  
看下它们的构造函数:   
`Camera1`   
``` java
Camera1(Callback callback, PreviewImpl preview) {
   super(callback, preview);
   preview.setCallback(new PreviewImpl.Callback() {
      @Override
      public void onSurfaceChanged() {
            if (mCamera != null) {
               setUpPreview();
               adjustCameraParameters();
            }
      }
   });
}
```
增加了 Callback，当SurfaceChanged的时候，设置预览和 Camera 参数。  
`Camera2`
``` java
Camera2(Callback callback, PreviewImpl preview, Context context) {
   super(callback, preview);
   mCameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
   mPreview.setCallback(new PreviewImpl.Callback() {
      @Override
      public void onSurfaceChanged() {
            startCaptureSession();
      }
   });
}
```
1.初始化了`CameraManager`  
2.增加Callback，当SurfaceChanged的时候，对CaptureSession进行设置  

`Camera2Api23`继承了`Camera2`，重用了`Camera2`的构造。  
Camera1和 Camera2的实现和使用流程，可以参考前面的几篇文章，接下来介绍`CameraView`的几个重要方法的实现。
### start
``` java
public void start() {
   if (!mImpl.start()) {
      //store the state ,and restore this state after fall back o Camera1
      Parcelable state = onSaveInstanceState();
      // Camera2 uses legacy hardware layer; fall back to Camera1
      mImpl = new Camera1(mCallbacks, createPreviewImpl(getContext()));
      onRestoreInstanceState(state);
      mImpl.start();
   }
}
```
开启相机，首先判断是否可以开启，如果不能开启，默认会使用 Camera1。
#### Camera1.start
``` java
boolean start() {
   //1. 选择摄像头
   chooseCamera();
   //2. 打开摄像头
   openCamera();
   //3. 设置预览
   if (mPreview.isReady()) {
      setUpPreview();
   }
   mShowingPreview = true;
   //4. 开始预览
   mCamera.startPreview();
   return true;
}
```
具体的流程，可以参考[Camera1使用](/2020/01/17/AndroidCamera-UsingCamera1/)，这里介绍几个重要的方法。   
1.`chooseCamera`会遍历所有摄像头，然后根据`CameraView`初始化时传入的值进行对比，默认是`FACING_BACK`后置摄像头。  
2.`openCamera`方法，这里具体展开介绍下：
``` java
private void openCamera() {
   if (mCamera != null) {
      releaseCamera();
   }
   //1. 打开摄像头
   mCamera = Camera.open(mCameraId);
   mCameraParameters = mCamera.getParameters();
   // Supported preview sizes
   mPreviewSizes.clear();
   //2. 获取所有支持的预览尺寸
   for (Camera.Size size : mCameraParameters.getSupportedPreviewSizes()) {
      Log.d("DEBUG", "###### SupportedPreviewSizes: width=" + size.width + ", height="
               + size.height);
      mPreviewSizes.add(new Size(size.width, size.height));
   }
   // Supported picture sizes;
   mPictureSizes.clear();
   //3. 获取所有支持的拍照尺寸
   for (Camera.Size size : mCameraParameters.getSupportedPictureSizes()) {
      Log.d("DEBUG", "###### SupportedPictureSizes: width=" + size.width + ", height="
               + size.height);
      mPictureSizes.add(new Size(size.width, size.height));
   }
   // AspectRatio
   if (mAspectRatio == null) {
      mAspectRatio = Constants.DEFAULT_ASPECT_RATIO;
   }
   //4. 设置预览比例
   adjustCameraParameters();
   mCamera.setDisplayOrientation(calcDisplayOrientation(mDisplayOrientation));
   mCallback.onCameraOpened();
}
```
3.`setUpPreview`设置预览
``` java
void setUpPreview() {
   try {
      if (mPreview.getOutputClass() == SurfaceHolder.class) {
            mCamera.setPreviewDisplay(mPreview.getSurfaceHolder());
      } else {
            mCamera.setPreviewTexture((SurfaceTexture) mPreview.getSurfaceTexture());
      }
   } catch (IOException e) {
      throw new RuntimeException(e);
   }
}
```
根据mPreview类型的不同，使用`SurfaceView`预览或者`TextureView`预览。
4.`mCamera.startPreview()`开启预览

#### Camera2.start
``` java
boolean start() {
   //1. 选择摄像头，默认后置摄像头
   if (!chooseCameraIdByFacing()) {
      return false;
   }
   //2. 设置预览尺寸，预览尺寸比例，拍照尺寸，其他相关设置
   collectCameraInfo();
   //3. 初始化 ImageReader，并设置回调
   prepareImageReader();
   //4. 打开摄像头
   startOpeningCamera();
   return true;
}
```
具体流程可以参考[Camera2使用](/2020/01/19/AndroidCamera-UsingCamera2/)，这里只介绍重要方法说明。     
1.`chooseCameraIdByFacing`遍历支持的摄像头列表，根据条件筛选获取到指定的摄像头，默认后置摄像头。     
2.`collectCameraInfo`方法，获取所有支持预览尺寸，和拍照尺寸，并获取支持的预览尺寸比例   
3.`prepareImageReader`方法，初始化ImageReader  
``` java
private void prepareImageReader() {
   if (mImageReader != null) {
      mImageReader.close();
   }
   Size largest = mPictureSizes.sizes(mAspectRatio).last();
   mImageReader = ImageReader.newInstance(largest.getWidth(), largest.getHeight(),
            ImageFormat.JPEG, /* maxImages */ 2);
   mImageReader.setOnImageAvailableListener(mOnImageAvailableListener, null);
}
```
设置输出格式为 JPEG，并增加`ImageAvailableListener`回调监听。  
4.`startOpeningCamera`方法，打开摄像头，并设置`CameraDevice.StateCallback`监听  
``` java
private void startOpeningCamera() {
   try {
      mCameraManager.openCamera(mCameraId, mCameraDeviceCallback, null);
   } catch (CameraAccessException e) {
      throw new RuntimeException("Failed to open camera: " + mCameraId, e);
   }
}

private final CameraDevice.StateCallback mCameraDeviceCallback
      = new CameraDevice.StateCallback() {

   @Override
   public void onOpened(@NonNull CameraDevice camera) {
      mCamera = camera;
      mCallback.onCameraOpened();
      //相机打开，开启预览画面
      startCaptureSession();
   }

   @Override
   public void onClosed(@NonNull CameraDevice camera) {
      mCallback.onCameraClosed();
   }

   @Override
   public void onDisconnected(@NonNull CameraDevice camera) {
      mCamera = null;
   }

   @Override
   public void onError(@NonNull CameraDevice camera, int error) {
      Log.e(TAG, "onError: " + camera.getId() + " (" + error + ")");
      mCamera = null;
   }

};
```
监听相机打开后，开启预览画面，`startCaptureSession`
``` java
void startCaptureSession() {
   if (!isCameraOpened() || !mPreview.isReady() || mImageReader == null) {
      return;
   }
   //1. 选择最合适的预览尺寸
   Size previewSize = chooseOptimalSize();
   mPreview.setBufferSize(previewSize.getWidth(), previewSize.getHeight());
   Surface surface = mPreview.getSurface();
   try {
      //2. 创建预览请求
      mPreviewRequestBuilder = mCamera.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
      //3. 请求管理 target surface
      mPreviewRequestBuilder.addTarget(surface);
      //4. 创建CaptureSession，并增加Session监听
      mCamera.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
               mSessionCallback, null);
   } catch (CameraAccessException e) {
      throw new RuntimeException("Failed to start camera session");
   }
}
```
监听`CameraCaptureSession`状态的回调声明：
``` java
private final CameraCaptureSession.StateCallback mSessionCallback
      = new CameraCaptureSession.StateCallback() {

   @Override
   public void onConfigured(@NonNull CameraCaptureSession session) {
      if (mCamera == null) {
            return;
      }
      mCaptureSession = session;
      updateAutoFocus();
      updateFlash();
      try {
         //1. 开启预览，并设置监听回调
            mCaptureSession.setRepeatingRequest(mPreviewRequestBuilder.build(),
                  mCaptureCallback, null);
      } catch (CameraAccessException e) {
            Log.e(TAG, "Failed to start camera preview because it couldn't access camera", e);
      } catch (IllegalStateException e) {
            Log.e(TAG, "Failed to start camera preview.", e);
      }
   }

   @Override
   public void onConfigureFailed(@NonNull CameraCaptureSession session) {
      Log.e(TAG, "Failed to configure capture session.");
   }

   @Override
   public void onClosed(@NonNull CameraCaptureSession session) {
      if (mCaptureSession != null && mCaptureSession.equals(session)) {
            mCaptureSession = null;
      }
   }

};
```
### takePicture
``` java
public void takePicture() {
   mImpl.takePicture();
}
```
根据 api 选择不同的实现
#### Camera1.takePicture
``` java
void takePicture() {
   if (!isCameraOpened()) {
      throw new IllegalStateException(
               "Camera is not ready. Call start() before takePicture().");
   }
   //1. 判断是否自动对焦
   if (getAutoFocus()) {
      mCamera.cancelAutoFocus();
      mCamera.autoFocus(new Camera.AutoFocusCallback() {
            @Override
            public void onAutoFocus(boolean success, Camera camera) {
               //2. 拍照
               takePictureInternal();
            }
      });
   } else {
      //2.拍照
      takePictureInternal();
   }
}
```
真正执行拍照的方法`takePictureInternal`：
``` java
void takePictureInternal() {
   if (!isPictureCaptureInProgress.getAndSet(true)) {
      //1. 拍照增加回调
      mCamera.takePicture(null, null, null, new Camera.PictureCallback() {
            @Override
            public void onPictureTaken(byte[] data, Camera camera) {
               isPictureCaptureInProgress.set(false);
               //2. 把 data 传到上层的回调
               mCallback.onPictureTaken(data);
               camera.cancelAutoFocus();
               camera.startPreview();
            }
      });
   }
}
```
`takePicture`中增加的 jpeg 的`PictureCallback`回调，返回的data通过callback回调到上层
#### Camera2.takePicture
``` java
void takePicture() {
   //1.判断自动对焦
   if (mAutoFocus) {
      lockFocus();
   } else {
      //2. 拍照请求
      captureStillPicture();
   }
}

//设置对焦
private void lockFocus() {
   mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
            CaptureRequest.CONTROL_AF_TRIGGER_START);
   try {
      mCaptureCallback.setState(PictureCaptureCallback.STATE_LOCKING);
      mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, null);
   } catch (CameraAccessException e) {
      Log.e(TAG, "Failed to lock focus.", e);
   }
}

//PictureCaptureCallback
PictureCaptureCallback mCaptureCallback = new PictureCaptureCallback() {
   @Override
   public void onPrecaptureRequired() {
      mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER,
               CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER_START);
      setState(STATE_PRECAPTURE);
      try {
            mCaptureSession.capture(mPreviewRequestBuilder.build(), this, null);
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER,
                  CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER_IDLE);
      } catch (CameraAccessException e) {
            Log.e(TAG, "Failed to run precapture sequence.", e);
      }
   }

   @Override
   public void onReady() {
      //2. 拍照请求
      captureStillPicture();
   }
};
```
判断是否自动对焦，最后调用`captureStillPicture`方法进行拍照请求：
``` java
void captureStillPicture() {
   try {
      //1. 创建TEMPLATE_STILL_CAPTURE的Capture请求
      CaptureRequest.Builder captureRequestBuilder = mCamera.createCaptureRequest(
               CameraDevice.TEMPLATE_STILL_CAPTURE);
      //2. 添加target
      captureRequestBuilder.addTarget(mImageReader.getSurface());
      //3. 设置 AF mode
      captureRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
               mPreviewRequestBuilder.get(CaptureRequest.CONTROL_AF_MODE));
      //4.flash模式设置
      ...
      // Calculate JPEG orientation.
      //5. 计算拍照图片的方向
      @SuppressWarnings("ConstantConditions")
      int sensorOrientation = mCameraCharacteristics.get(
               CameraCharacteristics.SENSOR_ORIENTATION);
      captureRequestBuilder.set(CaptureRequest.JPEG_ORIENTATION,
               (sensorOrientation +
                        mDisplayOrientation * (mFacing == Constants.FACING_FRONT ? 1 : -1) +
                        360) % 360);
      // Stop preview and capture a still picture.
      //6. 停止预览
      mCaptureSession.stopRepeating();
      //7. 拍照
      mCaptureSession.capture(captureRequestBuilder.build(),
               new CameraCaptureSession.CaptureCallback() {
                  @Override
                  public void onCaptureCompleted(@NonNull CameraCaptureSession session,
                           @NonNull CaptureRequest request,
                           @NonNull TotalCaptureResult result) {
                        //8. 取消对焦
                        unlockFocus();
                  }
               }, null);
   } catch (CameraAccessException e) {
      Log.e(TAG, "Cannot capture a still picture.", e);
   }
}

//取消对焦，重新设置预览
void unlockFocus() {
   mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
            CaptureRequest.CONTROL_AF_TRIGGER_CANCEL);
   try {
      mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, null);
      updateAutoFocus();
      updateFlash();
      mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
               CaptureRequest.CONTROL_AF_TRIGGER_IDLE);
      mCaptureSession.setRepeatingRequest(mPreviewRequestBuilder.build(), mCaptureCallback,
               null);
      mCaptureCallback.setState(PictureCaptureCallback.STATE_PREVIEW);
   } catch (CameraAccessException e) {
      Log.e(TAG, "Failed to restart camera preview.", e);
   }
}
```
拍照后真正的数据回调，是在初始化 ImageReader 的`OnImageAvailableListener`中
``` java
private final ImageReader.OnImageAvailableListener mOnImageAvailableListener
            = new ImageReader.OnImageAvailableListener() {
   @Override
   public void onImageAvailable(ImageReader reader) {
      try (Image image = reader.acquireNextImage()) {
            Image.Plane[] planes = image.getPlanes();
            if (planes.length > 0) {
               ByteBuffer buffer = planes[0].getBuffer();
               byte[] data = new byte[buffer.remaining()];
               buffer.get(data);
               //1. 数据回调到上层
               mCallback.onPictureTaken(data);
            }
      }
   }
};
```
### stop
如果不用拍照，或者退出应用，调用 stop 方法
``` java
public void stop() {
   mImpl.stop();
}
```
#### Camera1.stop
``` java
void stop() {
   if (mCamera != null) {
      //1. 停止预览
      mCamera.stopPreview();
   }
   mShowingPreview = false;
   //2. 释放相机资源
   releaseCamera();
}

//释放相机资源
private void releaseCamera() {
   if (mCamera != null) {
      mCamera.release();
      mCamera = null;
      //1. 回调上层
      mCallback.onCameraClosed();
   }
}
```
#### Camera2.stop
``` java
void stop() {
   //1. 关闭 session
   if (mCaptureSession != null) {
      mCaptureSession.close();
      mCaptureSession = null;
   }
   
   //2. 关闭 Camera
   if (mCamera != null) {
      mCamera.close();
      mCamera = null;
   }
   //3. 关闭 ImageReader
   if (mImageReader != null) {
      mImageReader.close();
      mImageReader = null;
   }
}
```
CameraView源码相关的分析就结束了，该库很好的封装了 Camera1和 Camera2的使用，本篇文章对其中的重点方法进行了分析，讲解了它的实现原理，希望对大家有所帮助。

# 参考：
* [Android相机开发——CameraView源码解析](https://www.jianshu.com/p/0ac7234dcefc)
* [Android Camera 编程从入门到精通](https://www.jianshu.com/p/f63f296a920b)
* [Android——谷歌cameraview详解](https://blog.csdn.net/shanshui911587154/article/details/90290535)