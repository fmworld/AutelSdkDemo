# Autel SDK 开发环境集成
1) gradle引入SDK资源包(目前只支持Android Studio开发环境，暂不支持Eclipse)

在project的gradle文件中配置maven服务器路径：
```
allprojects {
	repositories {
        maven(){
            url "http://x.x.x.x/artifactory/autel-snapshot"
        }
    }
}
```
示例如图

![maven服务器配置](/gradle_maven_repo1.PNG)

引入仓库后，在开发模块的gradle文件中建立依赖
```
	compile "com.autel.starlink:autel-sdk:2.0.2@aar"
```
其中，2.0.2为所选用的版本，示例如图

![compile依赖](/autel_app_compile.PNG)

2) 初始化SDK

在需要使用SDK功能前，调用init函数初始化相关SDK功能：

``` 
	String appKey = "<SDK license should be input>";
    	Autel.init(this, appKey, new CallbackWithNoParam() {...});
```
Autel.init()函数的第一个参数是会被**长期持有**的Context对象，为避免**内存泄露**，建议使用Application的Context对象

例如在自定义Application中初始化SDK服务：

![SDK初始化](/autel_sdk_init.PNG)


**NOTE: appKey 为开发者在[Autel开发平台](http:www.baidu.com)申请的应用关联钥匙**

3) SDK 功能接口调用

SDK提供以下模块的功能服务：Album（相册）、Battery（电池）、Camera（相机）、DSP（图传）、Firmware（固件）、FlyController（飞行控制器）、Gimbal（云台）、Mission（任务）、RemoteController（遥控器）、Codec(视频解码)


用户可以通过Autel类提供的静态方法获取对应的接口，例如获取任务模块接口：
```
	MissionManager missonManager = Autel.getMissionManager();
```
或者通过相应模块类的静态方法获取对应的接口，例如获取任务模块接口：
```
	MissionManager missonManager = AModuleMission.missionManager();
```
## 相机模块
目前可用相机类型为R12、FLIR，获取相机服务的接口时，需要通过CameraManager监听相机状态，当相机连接成功时会返回当前的相机类型，但是需要向下手动转型
```
		autelCameraManager.setConnectStateListener(new CallbackWithTwoParams<CameraProduct, AutelBaseCamera>() {
            	@Override
            	public void onSuccess(CameraProduct data1, AutelBaseCamera data2) {
                    if (data1 == CameraProduct.FLIR_DUO) {
                        AutelFLIR flir = (AutelFLIR)data2;
                    } else if (data1 == CameraProduct.R12) {
                        AutelR12 r12 = (AutelR12)data2;
                    } else if (data1 == CameraProduct.UNKNOWN) {
                        
                    }

			//或者

			if (data2 instanceof AutelFLIR) {
                        AutelFLIR flir = (AutelFLIR)data2;
                    } else if (data2 instanceof AutelR12) {
                        AutelR12 r12 = (AutelR12)data2;
                    } else {
                        
                    }
                }
            }

            @Override
            public void onFailure(AutelError error) {
            }
        });
```

由于不同的相机支持的参数范围可能会不一样，此时需要获取当前相机下支持的参数范围，调用示例如下
```
	AutelBaseCamera camera;
	...
	CameraProduct currentCameraProduct = camera.getProduct();
	CameraAspectRatio[] array = currentCameraProduct.supportedAspectRatio();
```
当前支持的查询参数有：VideoResolutionAndFps、AspectRatio、WhiteBalanceType、ISO、ExposureMode；

**注意**

VideoResolutionAndFps具体的参数范围依赖于相机当前的视频标准VideoStandard，调用示例如下
```
	AutelR12 autelR12；
	VideoStandard videoStandard;
	...
	autelR12.getVideoStandard(new CallbackWithOneParam<VideoStandard>() {
                    @Override
                    public void onFailure(AutelError error) {
                    }

                    @Override
                    public void onSuccess(VideoStandard data) {
						videoStandard = data;
                    }
                });

	CameraProduct currentCameraProduct = autelR12.getProduct();
	currentCameraProduct.supportedVideoResolutionAndFps(videoStandard)
```
## 视频解码模块
视频解码提供了两种获取解码数据方式，一种是提供自定义View直接显示飞行器相机实时传送的数据，只需要引用com.autel.sdk.widget.AutelCodecView即可；
```
	//xml实例化 
	<com.autel.sdk.widget.AutelCodecView
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

	//或者代码中动态生成
	AutelCodecView autelCodecView = new AutelCodecView(this);
        content_layout.setVisibility(View.VISIBLE);
        content_layout.addView(autelCodecView);
	
```
或者使用视频解码服务提供的数据监听接口获取数据
```
	AutelCodec codec = Autel.getCodec();
	...
	codec.setCodecListener(new AutelCodecListener() {
            @Override
            public void onFrameStream(final boolean valid, byte[] videoBuffer, final int size, final long pts) {
            }

            @Override
            public void onCanceled() {
            }

            @Override
            public void onFailure(final AutelError error) {
            }
        });
```

**注意**

AutelCodecView提供了四个功能接口：setOverExposure(boolean enabled, int resId)、isOverExposureEnabled()、pause()、resume()

AutelCodec提供了setCodecListener和cancel两个接口，其中setCodecListener设置监听的同时开启视频解码，cancel用于取消视频解码

## 任务模块

任务模块目前支持三种任务：WaypointMission(航点任务)、OrbitMission(环绕任务)、FollowMission(跟随任务)，所有任务由任务管理器(地图纠偏)执行相关操作

任务管理器的具体操作有：prepare(准备)、start(开始)、pause(暂停)、resume(继续)，cancel(取消)、download(下载正在执行的任务)

环绕任务为例，相关操作代码如下：

生成环绕任务实例：
```
    	OrbitMission mOrbitMission = new OrbitMission();
    	mOrbitMission.lat = (float) autelLatLng.latitude;
    	mOrbitMission.lng = (float) autelLatLng.longitude;
    	mOrbitMission.finishReturnHeight = 20;
    	mOrbitMission.finishedAction = missionFinishedAction;
    	mOrbitMission.speed = 3;
    	mOrbitMission.round = 3;
    	mOrbitMission.radius = 10;
```
使用MissionManager来准备环绕任务mOrbitMission
```
	MissionManager myMissonManager = Autel.getMissionManager();
	myMissonManager.prepareMission(mOrbitMission, new CallbackWithOneParamProgress<Boolean>() {...});
```

开始执行环绕任务
```
	myMissonManager.startMission(new CallbackWithNoParam() {...});
```
暂停执行环绕任务
```
	myMissonManager.pauseMission(new CallbackWithNoParam() {...});
```
继续执行环绕任务
```
	myMissonManager.resumeMission(new CallbackWithNoParam() {...});
```
取消执行环绕任务
```
	myMissonManager.cancelMission(new CallbackWithNoParam() {...});
```
下载正在执行的任务
```
	myMissonManager.downloadMission(new CallbackWithOneParamProgress<AutelMission>() {...});
```
- **任务模块相关注意事项**

1) 由于任务需要采集GPS信息，GPS模块(手机或者飞行器)采集的数据和地图工具(Google地图、高德地图)输出的数据相比较,针对**中国大陆地区**的GPS信息可能存在**坐标系偏差**，如果不做地图纠偏处理，在执行飞行任务时，**会有500米左右的位置偏差**

2) 任何任务都需要任务管理器**准备完成**后(下载的正在执行任务除外，因为已经在执行中)，才能有效执行
```
	myMissonManager.prepareMission(mOrbitMission, new CallbackWithOneParamProgress<Boolean>() {...});
```

3) 下载正在执行的任务，回调结果返回的是实际执行任务的**父类对象AutelMission**，需要开发者手动**向下转型**
```
	myMissonManager.downloadMission(new CallbackWithOneParamProgress<AutelMission>() {
		@Override
            	public void onProgress(float v) {
			}

            	@Override
            	public void onSuccess(AutelMission autelMission) {
            		if(autelMission instanceof OrbitMission){
				OrbitMission downloadMission = (OrbitMission)autelMission;
			}
            	}

            	@Override
            	public void onFailure(AutelError autelError) {
                           
            	}
    });

```
4) 飞行器只有执行**航点任务**和**环绕任务**时会反馈任务实时信息，跟随任务不会反馈实时信息，针对实时信息监听接口，获取的数据对象需要手动**向下转型**
```
	myMissonManager.setRealTimeInfoListener(new CallbackWithTwoParams<CurrentMissionState, RealTimeInfo>() {
            	@Override
            	public void onSuccess(CurrentMissionState currentMissionState, RealTimeInfo realTimeInfo) {
                	if(realTimeInfo instanceof OrbitRealTimeInfo){
					
			}else if(realTimeInfo instanceof WaypointRealTimeInfo){
			}
            	}

            	@Override
            	public void onFailure(AutelError autelError) {
            	}
        });
```

5) 跟随任务需要实时传递被跟踪物体的坐标给飞行器，此操作不由任务管理器完成，需要被执行的跟随任务自身**调用位置更新接口**，跟随任务实例化对象只能通过**静态方法**生成
```
	FollowMission followMission = FollowMission.create();
    	followMission.location = mLocation;
    	followMission.finishedAction = missionFinishedAction;
    	followMission.finishReturnHeight = 20;
	...
	/**
	 * 需要更新被跟踪物体位置坐标时，由被执行任务调用位置更新接口
	 */
	followMission.update(mLocation);
```

## 相册模块
相册模块提供了相机媒体资源文件的元信息管理，信息包括：原图地址，小缩略图地址，大缩略图地址，创建时间，文件大小；
```
public interface MediaInfo {

    long getFileSize();

    String getFileTimeString();

    String getSmallThumbnail();

    String getLargeThumbnail();

    String getOriginalMedia();
}
```

**注意**

getVideoResolutionFromLocalFile和getVideoResolutionFromHttpHeader用于获取相机中的视频资源的分辨率大小，并不能适用于图片资源；
getVideoResolutionFromLocalFile输入的文件对象为从相机中下载的视频资源文件；
```	
	AModuleAlbum autelAlbum = AModuleAlbum.album();
	//file是从相机中下载到手机中的视频资源
	VideoResolutionAndFps data = autelAlbum.getVideoResolutionFromLocalFile(file);
```
