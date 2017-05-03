# Autel SDK 开发环境集成
1) gradle引入SDK资源包(目前只支持Android Studio开发环境，暂不支持Eclipse)

在project的gradle文件中配置maven服务器路径：
```
allprojects {
	repositories {
        	maven(){
            		url "http://10.240.12.2:8081/artifactory/autel-snapshot"
        	}
    	}
}
```
示例如图

![maven服务器配置](/gradle_maven_repo.PNG)

2) 初始化SDK

在需要使用SDK功能前，调用init函数初始化相关SDK功能：

``` 
	String appKey = "<SDK license should be input>";
    	Autel.init(this, appKey, new CallbackWithNoParam() {...});

```
**Autel.init()函数的第一个参数是会被长期持有的Context对象，为避免内存泄露，建议使用Application的Context对象**

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


