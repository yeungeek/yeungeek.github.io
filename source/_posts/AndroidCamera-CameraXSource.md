---
title: Android Camera-CameraX源码分析
date: 2020-02-28 21:35:11
tags:
   - Camera2
   - CameraX
categories:
   - Camera 
---
在前面一篇文章中，已经介绍了如何使用 CameraX，这篇文章就分析下 CameraX 主要流程的源码。
<!-- more -->
本篇分析的源码版本是1.0.0-alpha06，目前最新的 CameraX 版本是[1.0.0-alpha10](https://developer.android.com/jetpack/androidx/releases/camera)。  
# 引用
在 build.gradle 中声明
``` gradle
def camerax_version = "1.0.0-alpha06"
implementation "androidx.camera:camera-core:${camerax_version}"
implementation "androidx.camera:camera-camera2:${camerax_version}"

def camerax_view_version = "1.0.0-alpha03"
def camerax_ext_version = "1.0.0-alpha03"
//other
// If you to use the Camera View class
implementation "androidx.camera:camera-view:$camerax_view_version"
// If you to use Camera Extensions
implementation "androidx.camera:camera-extensions:$camerax_ext_version"
```
* camera-core：Camera核心库，设计架构的实现
* camera-camera2：Camera2的配置和操作封装
* camera-view：自定义的 CameraView 组件
* camera-extensions：Camera的扩展，用于访问设备专属供应商效果（例如散景、HDR 及其他功能）的 API

其中`camera-core`和`camera-camera2`是必须使用的库，使用该库，可以轻松地使用Camera2 API的功能
# CameraX结构
首先看下 CameraX 的属性：
``` java
private static final CameraX INSTANCE = new CameraX();
final CameraRepository mCameraRepository = new CameraRepository();
private final AtomicBoolean mInitialized = new AtomicBoolean(false);
private final UseCaseGroupRepository mUseCaseGroupRepository = new UseCaseGroupRepository();
private final ErrorHandler mErrorHandler = new ErrorHandler();
private CameraFactory mCameraFactory;
private CameraDeviceSurfaceManager mSurfaceManager;
private UseCaseConfigFactory mDefaultConfigFactory;
private Context mContext;
```
主要看其中几个重要属性： 
* CameraRepository：Camera仓库，保存可用 Camera 的列表
* UseCaseGroupRepository：UseCaseGroupLifecycleController实例仓库，每个UseCaseGroupLifecycleController都与一个LifecycleOwner相关联，该LifecycleOwner调节组中所有用例共享的通用生命周期
* CameraFactory：Camera抽象工厂，Camera2CameraFactory是具体的实现类
* CameraDeviceSurfaceManager：Camera设备与对应数据流管理，具体实现是Camera2DeviceSurfaceManager
* UseCaseConfigFactory：UseCase配置工厂

CameraX主要使用`UseCase`的概念与相机设备进行交互，目前提供的`UseCase`：
* 预览(Preview)
* 图片拍摄(ImageCapture)
* 图片分析(ImageAnalysis)

## CameraX初始化
![GXwNU1.png](https://s1.ax1x.com/2020/04/13/GXwNU1.png)
### Camera2Initializer
CameraX初始化方法：init
``` java
public static void init(Context context, @NonNull AppConfig appConfig) {
   INSTANCE.initInternal(context, appConfig);
}
```
init 是通过ContentProvier配置初始化，具体实现类`Camera2Initializer`
``` java
public final class Camera2Initializer extends ContentProvider {
    private static final String TAG = "Camera2Initializer";

    @Override
    public boolean onCreate() {
        Log.d(TAG, "CameraX initializing with Camera2 ...");

        CameraX.init(getContext(), Camera2AppConfig.create(getContext()));
        return false;
    }
    ...
}
```
在`AndroidMainifest.xml`会自动生成provider配置，ContentProvider的OnCreate调用比Applicantion的 onCreate调用更早。
``` xml
<provider
   android:name="androidx.camera.camera2.impl.Camera2Initializer"
   android:exported="false"
   android:multiprocess="true"
   android:authorities="${applicationId}.camerax-init"
   android:initOrder="100" />
```
### Camera2AppConfig
init 方法传入的 AppConfig 的 create：
``` java
public static AppConfig create(Context context) {
   // Create the camera factory for creating Camera2 camera objects
   CameraFactory cameraFactory = new Camera2CameraFactory(context);

   // Create the DeviceSurfaceManager for Camera2
   CameraDeviceSurfaceManager surfaceManager = new Camera2DeviceSurfaceManager(context);

   // Create default configuration factory
   ExtendableUseCaseConfigFactory configFactory = new ExtendableUseCaseConfigFactory();
   configFactory.installDefaultProvider(
            ImageAnalysisConfig.class, new ImageAnalysisConfigProvider(cameraFactory, context));
   configFactory.installDefaultProvider(
            ImageCaptureConfig.class, new ImageCaptureConfigProvider(cameraFactory, context));
   configFactory.installDefaultProvider(
            VideoCaptureConfig.class, new VideoCaptureConfigProvider(cameraFactory, context));
   configFactory.installDefaultProvider(
            PreviewConfig.class, new PreviewConfigProvider(cameraFactory, context));

   AppConfig.Builder appConfigBuilder =
            new AppConfig.Builder()
                  .setCameraFactory(cameraFactory)
                  .setDeviceSurfaceManager(surfaceManager)
                  .setUseCaseConfigFactory(configFactory);

   return appConfigBuilder.build();
}
```
通过 AppConfig.Builder 进行构建，CameraX中的默认属性都在这里初始化。后面具体讲到某个 UseCase 的时候，详细分析下具体的ConfigProvider
### CameraX.initInternal
CameraX真正初始化方法：`initInternal`
``` java
private void initInternal(Context context, AppConfig appConfig) {
   if (mInitialized.getAndSet(true)) {
      return;
   }

   mContext = context.getApplicationContext();
   mCameraFactory = appConfig.getCameraFactory(null);
   if (mCameraFactory == null) {
      throw new IllegalStateException(
               "Invalid app configuration provided. Missing CameraFactory.");
   }

   mSurfaceManager = appConfig.getDeviceSurfaceManager(null);
   if (mSurfaceManager == null) {
      throw new IllegalStateException(
               "Invalid app configuration provided. Missing CameraDeviceSurfaceManager.");
   }

   mDefaultConfigFactory = appConfig.getUseCaseConfigRepository(null);
   if (mDefaultConfigFactory == null) {
      throw new IllegalStateException(
               "Invalid app configuration provided. Missing UseCaseConfigFactory.");
   }

   mCameraRepository.init(mCameraFactory);
}
```
直接从 AppConfig 中获取到具体实例，mCameraFactory对应的实例是`Camera2CameraFactory`，mCameraRepository.init(mCameraFactory)进行 Camera 相关的初始化
### CameraRepository.init
``` java
public void init(CameraFactory cameraFactory) {
   synchronized (mCamerasLock) {
      try {
            Set<String> camerasList = cameraFactory.getAvailableCameraIds();
            for (String id : camerasList) {
               Log.d(TAG, "Added camera: " + id);
               mCameras.put(id, cameraFactory.getCamera(id));
            }
      ...
   }
}
```
`getAvailableCameraIds`获取可用 Camera Id列表，Camera2CameraFactory的`getCamera`真正初始化Camera
``` java
public BaseCamera getCamera(@NonNull String cameraId) {
   Camera camera = new Camera(mCameraManager, cameraId,
            mAvailabilityRegistry.getAvailableCameraCount(), sHandler);
   mAvailabilityRegistry.registerCamera(camera);
   return camera;
}
```
通过`CameraAvailabilityRegistry`的`registerCamera`方法进行Camera注册   
到此为止，CameraX 相关属性就初始化完成了

## bindToLifecycle

从第一个`UseCase`预览(preview)来讲解CameraX 生命周期过程，以及数据传输流程。  
前面一篇文章已经讲解过 CameraX 的使用，其中预览(preivew)，会先声明`PreviewConfig`,通过 config 生成`Preivew`，`preview.setOnPreviewOutputUpdateListener`设置监听Camera数据流。这一系列流程能够实现，主要通过`CameraX.bindToLifecycle`实现     
具体流程：   
![GXRjuq.png](https://s1.ax1x.com/2020/04/13/GXRjuq.png)

``` java
public static void bindToLifecycle(LifecycleOwner lifecycleOwner, UseCase... useCases) {
   Threads.checkMainThread();
   UseCaseGroupLifecycleController useCaseGroupLifecycleController =
            INSTANCE.getOrCreateUseCaseGroup(lifecycleOwner);
   UseCaseGroup useCaseGroupToBind = useCaseGroupLifecycleController.getUseCaseGroup();

   Collection<UseCaseGroupLifecycleController> controllers =
            INSTANCE.mUseCaseGroupRepository.getUseCaseGroups();
   //检查UseCase 只能在一个lifecycle上
   for (UseCase useCase : useCases) {
      for (UseCaseGroupLifecycleController controller : controllers) {
            UseCaseGroup useCaseGroup = controller.getUseCaseGroup();
            if (useCaseGroup.contains(useCase) && useCaseGroup != useCaseGroupToBind) {
               throw new IllegalStateException(
                        String.format(
                              "Use case %s already bound to a different lifecycle.",
                              useCase));
            }
      }
   }

   //onBind监听回调
   for (UseCase useCase : useCases) {
      useCase.onBind();
   }

   calculateSuggestedResolutions(lifecycleOwner, useCases);

   for (UseCase useCase : useCases) {
      useCaseGroupToBind.addUseCase(useCase);
      for (String cameraId : useCase.getAttachedCameraIds()) {
            attach(cameraId, useCase);
      }
   }

   useCaseGroupLifecycleController.notifyState();
}
```
### UseCaseGroupLifecycleController
创建 UseCaseGroupLifecycleController，UseCaseGroup控制器，通过Lifecycle组件进行 start 和 stop 操作
``` java
UseCaseGroupLifecycleController useCaseGroupLifecycleController =
            INSTANCE.getOrCreateUseCaseGroup(lifecycleOwner);
...
private UseCaseGroupLifecycleController getOrCreateUseCaseGroup(LifecycleOwner lifecycleOwner) {
   return mUseCaseGroupRepository.getOrCreateUseCaseGroup(
            lifecycleOwner, new UseCaseGroupRepository.UseCaseGroupSetup() {
               @Override
               public void setup(UseCaseGroup useCaseGroup) {
                  useCaseGroup.setListener(mCameraRepository);
               }
            });
}
```
通过UseCaseGroupRepository创建UseCaseGroupLifecycleController
``` java
UseCaseGroupLifecycleController getOrCreateUseCaseGroup(
            LifecycleOwner lifecycleOwner, UseCaseGroupSetup groupSetup) {
   UseCaseGroupLifecycleController useCaseGroupLifecycleController;
   synchronized (mUseCasesLock) {
      //如果有缓存，则直接返回，否则进行创建
      useCaseGroupLifecycleController = mLifecycleToUseCaseGroupControllerMap.get(
               lifecycleOwner);
      if (useCaseGroupLifecycleController == null) {
            useCaseGroupLifecycleController = createUseCaseGroup(lifecycleOwner);
            groupSetup.setup(useCaseGroupLifecycleController.getUseCaseGroup());
      }
   }
   return useCaseGroupLifecycleController;
}
...
 private UseCaseGroupLifecycleController createUseCaseGroup(LifecycleOwner lifecycleOwner) {
   ...
   // Need to add observer before creating UseCaseGroupLifecycleController to make sure
   // UseCaseGroups can be stopped before the latest active one is started.
   lifecycleOwner.getLifecycle().addObserver(createLifecycleObserver());
   UseCaseGroupLifecycleController useCaseGroupLifecycleController =
            new UseCaseGroupLifecycleController(lifecycleOwner.getLifecycle());
   //创建后，放入 map 缓存
   synchronized (mUseCasesLock) {
      mLifecycleToUseCaseGroupControllerMap.put(lifecycleOwner,
               useCaseGroupLifecycleController);
   }
   return useCaseGroupLifecycleController;
}
```
创建UseCaseGroupLifecycleController，并增加Lifecycle生命周期控制：
``` java
UseCaseGroupLifecycleController(Lifecycle lifecycle) {
   this(lifecycle, new UseCaseGroup());
}

/** Wraps an existing {@link UseCaseGroup} so it is controlled by lifecycle transitions. */
UseCaseGroupLifecycleController(Lifecycle lifecycle, UseCaseGroup useCaseGroup) {
   this.mUseCaseGroup = useCaseGroup;
   this.mLifecycle = lifecycle;
   //绑定Lifecycle
   lifecycle.addObserver(this);
}

@OnLifecycleEvent(Lifecycle.Event.ON_START)
public void onStart(LifecycleOwner lifecycleOwner) {
   synchronized (mUseCaseGroupLock) {
      mUseCaseGroup.start();
   }
}

@OnLifecycleEvent(Lifecycle.Event.ON_STOP)
public void onStop(LifecycleOwner lifecycleOwner) {
   synchronized (mUseCaseGroupLock) {
      mUseCaseGroup.stop();
   }
}

@OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
public void onDestroy(LifecycleOwner lifecycleOwner) {
   synchronized (mUseCaseGroupLock) {
      mUseCaseGroup.clear();
   }
}
```
上面的代码，增加了`ON_START`，`ON_STOP`，`ON_DESTROY`的生命周期监听  
### calculateSuggestedResolutions
根据传入的配置，生成各个UseCase的最佳解决方案。后面的代码会以`Preview`这个 UseCase 展开，其他 UseCase 代码逻辑类似。  
``` java
private static void calculateSuggestedResolutions(LifecycleOwner lifecycleOwner,
            UseCase... useCases) {
   // There will only one lifecycleOwner active. Therefore, only collect use cases belong to
   // same lifecycleOwner and calculate the suggested resolutions.
   ...
   // Collect new use cases for different camera devices
   for (UseCase useCase : useCases) {
      String cameraId = null;
      try {
            cameraId = getCameraWithCameraDeviceConfig(
                  (CameraDeviceConfig) useCase.getUseCaseConfig());
      } catch (CameraInfoUnavailableException e) {
            throw new IllegalArgumentException(
                  "Unable to get camera id for the camera device config.", e);
      }
   }
   ...
   // Get suggested resolutions and update the use case session configuration
   for (String cameraId : newCameraIdUseCaseMap.keySet()) {
      Map<UseCase, Size> suggestResolutionsMap =
               getSurfaceManager()
                        .getSuggestedResolutions(
                              cameraId,
                              originalCameraIdUseCaseMap.get(cameraId),
                              newCameraIdUseCaseMap.get(cameraId));

      for (UseCase useCase : newCameraIdUseCaseMap.get(cameraId)) {
            Size resolution = suggestResolutionsMap.get(useCase);
            Map<String, Size> suggestedCameraSurfaceResolutionMap = new HashMap<>();
            suggestedCameraSurfaceResolutionMap.put(cameraId, resolution);
            //更新配置
            useCase.updateSuggestedResolution(suggestedCameraSurfaceResolutionMap);
      }
   }
}
```
每个 UseCase 会去更新对应配置 updateSuggestedResolution->onSuggestedResolutionUpdated
``` java
public void updateSuggestedResolution(Map<String, Size> suggestedResolutionMap) {
   Map<String, Size> resolutionMap = onSuggestedResolutionUpdated(suggestedResolutionMap);
   ...
}
```
onSuggestedResolutionUpdated针对不同的 UseCase 有不同的实现，这里以`Preview`为例
``` java
protected Map<String, Size> onSuggestedResolutionUpdated(
            Map<String, Size> suggestedResolutionMap) {
   //获取前面配置的 config
   PreviewConfig config = (PreviewConfig) getUseCaseConfig();
   String cameraId = getCameraIdUnchecked(config);
   Size resolution = suggestedResolutionMap.get(cameraId);
   ...
   //设置 config
   updateConfigAndOutput(config, resolution);
   return suggestedResolutionMap;
}
...
private void updateConfigAndOutput(PreviewConfig config, Size resolution) {
   String cameraId = getCameraIdUnchecked(config);
   //初始化pipeline
   mSessionConfigBuilder = createPipeline(config, resolution);
   attachToCamera(cameraId, mSessionConfigBuilder.build());
   updateOutput(mSurfaceTextureHolder.getSurfaceTexture(), resolution);
}
```
### Preview.createPipeline
创建Preview管道，通过 PreviewConfig 的配置，创建对应的显示Surface和SessionConfig
``` java
SessionConfig.Builder createPipeline(PreviewConfig config, Size resolution) {
   Threads.checkMainThread();
   SessionConfig.Builder sessionConfigBuilder = SessionConfig.Builder.createFrom(config);

   final CaptureProcessor captureProcessor = config.getCaptureProcessor(null);
   //扩展的 extensions实现
   if (captureProcessor != null) {
      CaptureStage captureStage = new CaptureStage.DefaultCaptureStage();
      // TODO: To allow user to use an Executor for the processing.
      ...
   } else {
      final ImageInfoProcessor processor = config.getImageInfoProcessor(null);
      if (processor != null) {
            sessionConfigBuilder.addCameraCaptureCallback(new CameraCaptureCallback() {
               @Override
               public void onCaptureCompleted(
                        @NonNull CameraCaptureResult cameraCaptureResult) {
                  super.onCaptureCompleted(cameraCaptureResult);
                  if (processor.process(
                           new CameraCaptureResultImageInfo(cameraCaptureResult))) {
                        notifyUpdated();
                  }
               }
            });
      }
      //默认的 Surface
      CheckedSurfaceTexture checkedSurfaceTexture = new CheckedSurfaceTexture(resolution);

      mSurfaceTextureHolder = checkedSurfaceTexture;
      sessionConfigBuilder.addSurface(checkedSurfaceTexture);
   }
   ...
}
```
这里就可以看到我们熟悉的味道，在Camera2中 用到的`Surface`，`Session`相关配置，后面会用到相关配置。    
在`CheckedSurfaceTexture`中会创建`FixedSizeSurfaceTexture`用来显示图像。  
### Preview.updateOutput
增加数据的监听
``` java
void updateOutput(SurfaceTexture surfaceTexture, Size resolution) {
   PreviewConfig useCaseConfig = (PreviewConfig) getUseCaseConfig();
   ...
   PreviewOutput newOutput =
            PreviewOutput.create(surfaceTexture, resolution, relativeRotation);

   // Only update the output if something has changed
   if (!Objects.equals(mLatestPreviewOutput, newOutput)) {
      SurfaceTexture oldTexture =
               (mLatestPreviewOutput == null)
                        ? null
                        : mLatestPreviewOutput.getSurfaceTexture();
      OnPreviewOutputUpdateListener outputListener = getOnPreviewOutputUpdateListener();
      ...

      if (outputListener != null) {
            mSurfaceDispatched = true;
            updateListener(outputListener, newOutput);
      }
   }
}
```
根据Preview设置的setOnPreviewOutputUpdateListener，获取到对应的Listener，通过`updateListener`方法回调数据。
``` java
private void updateListener(OnPreviewOutputUpdateListener listener, PreviewOutput output) {
   ...
   mOutputUpdateExecutor.execute(() -> listener.onUpdated(output));
   ...
}
```
### notifyState
调用UseCaseGroupLifecycleController的notifyState，激活 UseCase 状态，在UseCaseGroupLifecycleController中有增加生命周期的监听，在`ON_START`状态会调用`mUseCaseGroup.start`方法。
``` java
void notifyState() {
   synchronized (mUseCaseGroupLock) {
      if (mLifecycle.getCurrentState().isAtLeast(State.STARTED)) {
            mUseCaseGroup.start();
      }
      for (UseCase useCase : mUseCaseGroup.getUseCases()) {
            useCase.notifyState();
      }
   }
}
```
### UseCaseGroup.start
``` java
void start() {
   synchronized (mListenerLock) {
      if (mListener != null) {
            mListener.onGroupActive(this);
      }
      mIsActive = true;
   }
}
```
启动 start 状态，调用`CameraRepository`的`onGroupActive`方法：
``` java
public void onGroupActive(UseCaseGroup useCaseGroup) {
   synchronized (mCamerasLock) {
      Map<String, Set<UseCase>> cameraIdToUseCaseMap = useCaseGroup.getCameraIdToUseCaseMap();
      for (Map.Entry<String, Set<UseCase>> cameraUseCaseEntry :
               cameraIdToUseCaseMap.entrySet()) {
            BaseCamera camera = getCamera(cameraUseCaseEntry.getKey());
            attachUseCasesToCamera(camera, cameraUseCaseEntry.getValue());
      }
   }
}
...
private void attachUseCasesToCamera(BaseCamera camera, Set<UseCase> useCases) {
   camera.addOnlineUseCase(useCases);
}
```
`camera.addOnlineUseCase`关联UseCase 和 Camera。
### Camera.addOnlineUseCase
``` java
public void addOnlineUseCase(@NonNull final Collection<UseCase> useCases) {
   if (useCases.isEmpty()) {
      return;
   }

   // Attaches the surfaces of use case to the Camera (prevent from surface abandon crash)
   // addOnlineUseCase could be called with duplicate use case, so we need to filter out
   // use cases that are either pending for addOnline or are already online.
   // It's ok for two thread to run here, since it‘ll do nothing if use case is already
   // pending.
   synchronized (mPendingLock) {
      for (UseCase useCase : useCases) {
            boolean isOnline = isUseCaseOnline(useCase);
            if (mPendingForAddOnline.contains(useCase) || isOnline) {
               continue;
            }

            notifyAttachToUseCaseSurfaces(useCase);
            mPendingForAddOnline.add(useCase);
      }
   }
   ...
   updateCaptureSessionConfig();
   resetCaptureSession(/*abortInFlightCaptures=*/false);

   if (mState == InternalState.OPENED) {
      openCaptureSession();
   } else {
      open();
   }

   updateCameraControlPreviewAspectRatio(useCases);
}
```
在addOnlineUseCase方法中，open会去打开Camera设备。
### Camera.open
``` java
public void open() {
   ...
   switch (mState) {
      case INITIALIZED:
            openCameraDevice();
            break;
      case CLOSING:
            setState(InternalState.REOPENING);
            // If session close has not yet completed, then the camera is still open. We
            // can move directly back into an OPENED state.
            // If session close is already complete, then the camera is closing. We'll reopen
            // the camera in the camera state callback.
            // If the camera device is currently in an error state, we need to close the
            // camera before reopening, so we cannot directly reopen.
            if (!isSessionCloseComplete() && mCameraDeviceError == ERROR_NONE) {
               Preconditions.checkState(mCameraDevice != null,
                        "Camera Device should be open if session close is not complete");
               setState(InternalState.OPENED);
               openCaptureSession();
            }
            break;
      default:
            Log.d(TAG, "open() ignored due to being in state: " + mState);
   }
}
...
void openCameraDevice() {
   // Check that we have an available camera to open here before attempting
   // to open the camera again.
   if (!mCameraAvailability.isCameraAvailable()) {
      Log.d(TAG, "No cameras available. Waiting for available camera before opening camera: "
               + mCameraId);
      setState(InternalState.PENDING_OPEN);
      return;
   } else {
      setState(InternalState.OPENING);
   }
   ...
   //真正打开相机
   mCameraManager.openCamera(mCameraId, mExecutor, createDeviceStateCallback());
   ...
}
```
接下来就是Camera2的预览流程
## Camera
CameraX封装了Camera2的标准预览流程，这些类都是在 CameraX 库中
![GXLFRP.png](https://s1.ax1x.com/2020/04/13/GXLFRP.png)
### CameraDevice.StateCallback
`openCameraDevice`的stateCallback
``` java
final class StateCallback extends CameraDevice.StateCallback {
   @Override
   public void onOpened(CameraDevice cameraDevice) {
      Log.d(TAG, "CameraDevice.onOpened(): " + cameraDevice.getId());
      mCameraDevice = cameraDevice;
      mCameraDeviceError = ERROR_NONE;
      switch (mState) {
            case CLOSING:
            case RELEASING:
               // No session should have yet been opened, so close camera directly here.
               Preconditions.checkState(isSessionCloseComplete());
               mCameraDevice.close();
               mCameraDevice = null;
               break;
            case OPENING:
            case REOPENING:
               setState(InternalState.OPENED);
               openCaptureSession();
               break;
            default:
               throw new IllegalStateException(
                        "onOpened() should not be possible from state: " + mState);
      }
   }
   ...
}
...
void openCaptureSession() {
   ...
   mCaptureSession.open(validatingBuilder.build(), mCameraDevice);
   ...
}
```
### CaptureSession.open
创建CaptureSession
``` java
void open(SessionConfig sessionConfig, CameraDevice cameraDevice)
      throws CameraAccessException, DeferrableSurface.SurfaceClosedException {
   synchronized (mStateLock) {
      switch (mState) {
            case UNINITIALIZED:
               throw new IllegalStateException(
                        "open() should not be possible in state: " + mState);
            case INITIALIZED:
               //Camera中传入的essionConfig，默认有TEMPLATE_PREVIEW的 surface 实现
               List<DeferrableSurface> surfaces = sessionConfig.getSurfaces();
               ...
               //状态更新
               notifySurfaceAttached();
               mState = State.OPENING;
               ...
               SessionConfigurationCompat sessionConfigCompat =
                        new SessionConfigurationCompat(
                              SessionConfigurationCompat.SESSION_REGULAR,
                              outputConfigList,
                              getExecutor(),
                              comboCallback);

               CaptureRequest captureRequest =
                        Camera2CaptureRequestBuilder.buildWithoutTarget(
                              captureConfigBuilder.build(),
                              cameraDevice);

               if (captureRequest != null) {
                  sessionConfigCompat.setSessionParameters(captureRequest);
               }
               //创建CaptureSession，CameraDeviceCompat根据 Android 版本有不同的实现
               CameraDeviceCompat.createCaptureSession(cameraDevice, sessionConfigCompat);

      ...
      }
   }
}  
```
在Camera2的使用中，CameraDevice的createCaptureSession可以创建预览画面，CameraX的CaptureSession很好的封装了这些实现。  
在`CaptureSession.open`传入的SessionConfig，是在`Camera2AppConfig.create`创建的时候生成
``` java
public static AppConfig create(Context context) {
   // Create the camera factory for creating Camera2 camera objects
   CameraFactory cameraFactory = new Camera2CameraFactory(context);
   // PreviewConfig 配置
   configFactory.installDefaultProvider(
            PreviewConfig.class, new PreviewConfigProvider(cameraFactory, context));

   AppConfig.Builder appConfigBuilder =
            new AppConfig.Builder()
                  .setCameraFactory(cameraFactory)
                  .setDeviceSurfaceManager(surfaceManager)
                  .setUseCaseConfigFactory(configFactory);

   return appConfigBuilder.build();
}
//PreviewConfigProvider配置getConfig
@Override
public PreviewConfig getConfig(LensFacing lensFacing) {
   PreviewConfig.Builder builder =
            PreviewConfig.Builder.fromConfig(Preview.DEFAULT_CONFIG.getConfig(lensFacing));
   // SessionConfig containing all intrinsic properties needed for Preview
   SessionConfig.Builder sessionBuilder = new SessionConfig.Builder();
   // createCaptureSession中的 preview 设置
   sessionBuilder.setTemplateType(CameraDevice.TEMPLATE_PREVIEW);

   // Add options to UseCaseConfig
   builder.setDefaultSessionConfig(sessionBuilder.build());
   builder.setSessionOptionUnpacker(Camera2SessionOptionUnpacker.INSTANCE);

   CaptureConfig.Builder captureBuilder = new CaptureConfig.Builder();
   captureBuilder.setTemplateType(CameraDevice.TEMPLATE_PREVIEW);
   builder.setDefaultCaptureConfig(captureBuilder.build());
   builder.setCaptureOptionUnpacker(Camera2CaptureOptionUnpacker.INSTANCE);
   ...
}
```
`CameraDeviceCompat.createCaptureSession`的CameraCaptureSession回调
``` java
final class StateCallback extends CameraCaptureSession.StateCallback {
   @Override
   public void onConfigured(@NonNull CameraCaptureSession session) {
      synchronized (mStateLock) {
            switch (mState) {
               case UNINITIALIZED:
               case INITIALIZED:
               case OPENED:
               case RELEASED:
                  throw new IllegalStateException(
                           "onConfigured() should not be possible in state: " + mState);
               case OPENING:
                  ...
                  // Issue capture request of enableSession if exists.
                  if (mSessionConfig != null) {
                        Config implOptions = mSessionConfig.getImplementationOptions();
                        CameraEventCallbacks eventCallbacks = new Camera2Config(
                              implOptions).getCameraEventCallback(
                              CameraEventCallbacks.createEmptyCallback());
                              //可配置CameraEventCallback的EnableSession回调
                        List<CaptureConfig> list =
                              eventCallbacks.createComboCallback().onEnableSession();
                        if (!list.isEmpty()) {
                           issueCaptureRequests(setupConfiguredSurface(list));
                        }
                  }
                  //
                  issueRepeatingCaptureRequests();
                  issueBurstCaptureRequest();
                  break;
               ...
            }
      }
   }
}
```
### CaptureSession.issueRepeatingCaptureRequests
开启Camera预览
``` java
void issueRepeatingCaptureRequests() {
   ...
   CaptureConfig captureConfig = mSessionConfig.getRepeatingCaptureConfig();
   ...
   // The override priority for implementation options
   // P1 CameraEventCallback onRepeating options
   // P2 SessionConfig options
   CaptureConfig.Builder captureConfigBuilder = CaptureConfig.Builder.from(captureConfig);

   //创建CaptureRequest
   CaptureRequest captureRequest = Camera2CaptureRequestBuilder.build(
            captureConfigBuilder.build(), mCameraCaptureSession.getDevice(),
            mConfiguredSurfaceMap);
   if (captureRequest == null) {
         Log.d(TAG, "Skipping issuing empty request for session.");
         return;
   }

   //设置Capture回调
   CameraCaptureSession.CaptureCallback comboCaptureCallback =
            createCamera2CaptureCallback(
                     captureConfig.getCameraCaptureCallbacks(),
                     mCaptureCallback);

   CameraCaptureSessionCompat.setSingleRepeatingRequest(mCameraCaptureSession,
            captureRequest, mExecutor, comboCaptureCallback);
}
```
CameraCaptureSessionCompat.setSingleRepeatingRequest 也是区分 Android 版本
``` java
private static CameraCaptureSessionCompatImpl chooseImplementation() {
   if (Build.VERSION.SDK_INT >= 28) {
      return new CameraCaptureSessionCompatApi28Impl();
   }

   return new CameraCaptureSessionCompatBaseImpl();
}
//CameraCaptureSessionCompatBaseImpl，和平时使用的一样
public int setSingleRepeatingRequest(@NonNull CameraCaptureSession captureSession,
      @NonNull CaptureRequest request, @NonNull Executor executor,
      @NonNull CameraCaptureSession.CaptureCallback listener) throws CameraAccessException {
   Preconditions.checkNotNull(captureSession);

   // Wrap the executor in the callback
   CameraCaptureSession.CaptureCallback cb =
            new CameraCaptureSessionCompat.CaptureCallbackExecutorWrapper(executor, listener);

   return captureSession.setRepeatingRequest(
            request, cb, MainThreadAsyncHandler.getInstance());
}
//CameraCaptureSessionCompatApi28Impl，新版本 API 有些变化
public int setSingleRepeatingRequest(@NonNull CameraCaptureSession captureSession,
      @NonNull CaptureRequest request, @NonNull Executor executor,
      @NonNull CameraCaptureSession.CaptureCallback listener) throws CameraAccessException {
   Preconditions.checkNotNull(captureSession);

   // Call through directly to executor API
   return captureSession.setSingleRepeatingRequest(request, executor, listener);
}
```
从Camera的开启到预览，以及读取各种配置，整个过程到此就完成了，接下来介绍如何拍照，这个流程相对来说比较简单
## ImageCapture.takePicture
拍照的流程：
![JZ52bd.png](https://s1.ax1x.com/2020/04/17/JZ52bd.png)
### sendImageCaptureRequest
创建 ImageCaptureRequest，设置cameraId、targetRatio、回调等
``` java
private void sendImageCaptureRequest(
      @Nullable Executor listenerExecutor, OnImageCapturedListener listener) {

   String cameraId = getCameraIdUnchecked(mConfig);

   // Get the relative rotation or default to 0 if the camera info is unavailable
   int relativeRotation = 0;
   try {
      CameraInfoInternal cameraInfoInternal = CameraX.getCameraInfo(cameraId);
      relativeRotation =
               cameraInfoInternal.getSensorRotationDegrees(
                        mConfig.getTargetRotation(Surface.ROTATION_0));
   } catch (CameraInfoUnavailableException e) {
      Log.e(TAG, "Unable to retrieve camera sensor orientation.", e);
   }

   Rational targetRatio = mConfig.getTargetAspectRatioCustom(null);
   targetRatio = ImageUtil.rotate(targetRatio, relativeRotation);

   mImageCaptureRequests.offer(
            new ImageCaptureRequest(relativeRotation, targetRatio, listenerExecutor, listener));
   if (mImageCaptureRequests.size() == 1) {
      issueImageCaptureRequests();
   }
}
```
### takePictureInternal
``` java
void issueImageCaptureRequests() {
   if (mImageCaptureRequests.isEmpty()) {
      return;
   }
   takePictureInternal();
}
...
//拍照流程
private void takePictureInternal() {
   //自定义 Future 调用链
   FutureChain.from(preTakePicture(state))
      .transformAsync{
         ...
         return ImageCapture.this.issueTakePicture(state);
      })
      .transformAsync{
         ...
         return ImageCapture.this.postTakePicture(state);
      })
      .addCallback(
         ...
         onTakePictureFinish(null);
      )
}
```
自定义了整个拍照工作流，通过`issueTakePicture`进行拍照，`postTakePicture`是拍照成功，释放资源，取消3A。下面重点看下`issueTakePicture`流程
### issueTakePicture
``` java
ListenableFuture<Void> issueTakePicture(TakePictureState state) {
   ...
   getCurrentCameraControl().submitCaptureRequests(captureConfigs);
   ...
}
```
通过`CameraControl`提交Capture 请求，`CameraControl`具体实现是`Camera2CameraControl`。
### submitCaptureRequests
``` java
public void submitCaptureRequests(@NonNull final List<CaptureConfig> captureConfigs) {
   mExecutor.execute(new Runnable() {
      @Override
      public void run() {
            submitCaptureRequestsInternal(captureConfigs);
      }
   });
}
...
void submitCaptureRequestsInternal(final List<CaptureConfig> captureConfigs) {
   mControlUpdateListener.onCameraControlCaptureRequests(captureConfigs);
   //mControlUpdateListener是Camera 的回调，onCameraControlCaptureRequests 真正实现在 Camera 中
}
//Camera.java
public void onCameraControlUpdateSessionConfig(@NonNull SessionConfig sessionConfig) {
   mCameraControlSessionConfig = sessionConfig;
   updateCaptureSessionConfig();
}
...
private void updateCaptureSessionConfig() {
      ...
         SessionConfig sessionConfig = validatingBuilder.build();
         mCaptureSession.setSessionConfig(sessionConfig);
      ...
   }
}
Camera 获取 Capture的SessionConfig，通过`CaptureSession`进行状态控制
```
### CaptureSession.setSessionConfig
``` java
void setSessionConfig(SessionConfig sessionConfig) {
   synchronized (mStateLock) {
      switch (mState) {
            case UNINITIALIZED:
               throw new IllegalStateException(
                        "setSessionConfig() should not be possible in state: " + mState);
            case INITIALIZED:
            case OPENING:
               mSessionConfig = sessionConfig;
               break;
            case OPENED:
               mSessionConfig = sessionConfig;

               if (!mConfiguredSurfaceMap.keySet().containsAll(sessionConfig.getSurfaces())) {
                  Log.e(TAG, "Does not have the proper configured lists");
                  return;
               }

               Log.d(TAG, "Attempting to submit CaptureRequest after setting");
               issueRepeatingCaptureRequests();
               break;
            case CLOSED:
            case RELEASING:
            case RELEASED:
               throw new IllegalStateException(
                        "Session configuration cannot be set on a closed/released session.");
      }
   }
}
```
在Camera的OPENED状态，则进行拍照流程
### issueRepeatingCaptureRequests
``` java
void issueRepeatingCaptureRequests() {
   ...
   CameraCaptureSession.CaptureCallback comboCaptureCallback =
                    createCamera2CaptureCallback(
                            captureConfig.getCameraCaptureCallbacks(),
                            mCaptureCallback);

   CameraCaptureSessionCompat.setSingleRepeatingRequest(mCameraCaptureSession,
                    captureRequest, mExecutor, comboCaptureCallback);
   ...
}
```
`CameraCaptureSessionCompat`根据 Android 版本有`CameraCaptureSessionCompatBaseImpl`和`CameraCaptureSessionCompatApi28Impl`两种实现，最终通过`CameraCaptureSession`实现真正的拍照。  
拍照完成后，通过最开始设置的 Listener 进行回调
### ImageCapture.createPipeline
在 Preview 那小节，讲解过 bindToLifecycle 流程，这里的`ImageCapture`也是一个`UseCase`。在CameraX中的`calculateSuggestedResolutions`方法，最终会调用到各个`UseCase`的`onSuggestedResolutionUpdated`方法。在`ImageCapture`的`onSuggestedResolutionUpdated`方法，通过`createPipeline`创建了拍照数据的回调
``` java
SessionConfig.Builder createPipeline(ImageCaptureConfig config,  Size resolution) {
   ...
   //和 Camera2的流程一致
   mProcessingImageResultThread = new HandlerThread("OnImageAvailableHandlerThread");
   mProcessingImageResultThread.start();
   mProcessingImageResultHandler = new Handler(mProcessingImageResultThread.getLooper());
   ...
   mImageReader.setOnImageAvailableListener(
      new ImageReaderProxy.OnImageAvailableListener() {
         @Override
         public void onImageAvailable(ImageReaderProxy imageReader) {
            ImageProxy image = null;
            try {
                  image = imageReader.acquireLatestImage();
            } catch (IllegalStateException e) {
                  Log.e(TAG, "Failed to acquire latest image.", e);
            } finally {
                  if (image != null) {
                     // Call the head request listener to process the captured image.
                     ImageCaptureRequest imageCaptureRequest;
                     if ((imageCaptureRequest = mImageCaptureRequests.peek()) != null) {
                        SingleCloseImageProxy wrappedImage = new SingleCloseImageProxy(
                                 image);
                        wrappedImage.addOnImageCloseListener(mOnImageCloseListener);
                        //ImageCaptureRequest设置 Listener
                        imageCaptureRequest.dispatchImage(wrappedImage);
                     } else {
                        // Discard the image if we have no requests.
                        image.close();
                     }
                  }
            }
         }
      },
      mProcessingImageResultHandler);
   ...
}
```
ImageReader设置了 Camera 数据会调用，并通过`ImageCaptureRequest`的`dispatchImage`方法进行分发
### ImageCaptureRequest.dispatchImage
``` java
void dispatchImage(final ImageProxy image) {
   try {
         mListenerExecutor.execute(new Runnable() {
            @Override
            public void run() {
               Size sourceSize = new Size(image.getWidth(), image.getHeight());
               if (ImageUtil.isAspectRatioValid(sourceSize, mTargetRatio)) {
                     image.setCropRect(
                           ImageUtil.computeCropRectFromAspectRatio(sourceSize,
                                    mTargetRatio));
               }
               //真正的回调
               mListener.onCaptureSuccess(image, mRotationDegrees);
            }
         });
   } catch (RejectedExecutionException e) {
         Log.e(TAG, "Unable to post to the supplied executor.");

         // Unable to execute on the supplied executor, close the image.
         image.close();
   }
}
```
`mListener`是一个封装Listener，在`ImageCapture`中实现
### Listener
Listener的关系图：
``` java
 +-----------------------+
 |                       |
 |ImageCapture.          |
 |OnImageCapturedListener|
 |                       |
 +-----------+-----------+
             |
             |
 +-----------v-----------+      +----------------------+
 |                       |      |                      |
 | ImageSaver.           |      | ImageCapture.        |
 | OnImageSavedListener  +------> OnImageSavedListener |
 |                       |      |                      |
 +-----------------------+      +----------------------+
```
`OnImageCapturedListener`的实现，其中通过`ImageSaver`设置的`OnImageSavedListener`回调到最上层的`OnImageSavedListener`
``` java
OnImageCapturedListener imageCaptureCallbackWrapper =
   new OnImageCapturedListener() {
      @Override
      public void onCaptureSuccess(ImageProxy image, int rotationDegrees) {
         CameraXExecutors.ioExecutor()
                  .execute(
                           new ImageSaver(
                                 image,
                                 saveLocation,
                                 rotationDegrees,
                                 metadata.isReversedHorizontal,
                                 metadata.isReversedVertical,
                                 metadata.location,
                                 executor,
                                 imageSavedListenerWrapper));
      }
   ...
   };
// ImageSaver是一个 Runnable，主要 run 的实现
final class ImageSaver implements Runnable {
   ...
   @Override
   public void run() {
      ...
      //图像处理
      ...
      if (saveError != null) {
         postError(saveError, errorMessage, exception);
      } else {
         postSuccess();
      }
   }
   ...
   private void postSuccess() {
      mExecutor.execute(new Runnable() {
         @Override
         public void run() {
            //最外层回调
            mListener.onImageSaved(mFile);
         }
      });
   }
}
```
整个拍照流程和数据回调就讲解完毕了。   
通过对 CameraX的 Preview 和 ImageCapture的分析，CameraX对Camera2进行完整的封装，统一参数配置，自动计算Resolution，简化Camera2的开发，并增加了生命周期控制，对外只暴露了简单接口。   
使用该库，只需要简单的几行代码就可以实现以前Camera2复杂的操作。

# 参考
* [CameraX 架构](https://developer.android.com/training/camerax/architecture)
* [Core Principles Behind CameraX Jetpack Library](https://medium.com/androiddevelopers/core-principles-behind-camerax-jetpack-library-8e8380f7604c)