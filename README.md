# 集成Autel SDK 开发环境
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
    Autel.init(this, appKey, new CallbackWithNoParam() {
          @Override
          public void onSuccess() {
          }

          @Override
          public void onFailure(AutelError error) {
          }
    });

```
例如在自定义Application中初始化SDK服务：

![SDK初始化](/autel_sdk_init.PNG)


**NOTE: appKey 为开发者在[Autel开发平台](http:www.baidu.com)申请的应用关联钥匙**

3) SDK 功能接口调用

SDK提供以下模块的功能服务：Album（相册）、Battery（电池）、Camera（相机）、DSP（图传）、Firmware（固件）、FlyController（飞行控制器）、Gimbal（云台）、Mission（任务）、RemoteController（遥控器）、Codec(视频解码)。

用户可以通过Autel类提供的静态方法获取对应的接口，例如获取任务模块接口：
```
	MissionManager missonManager = Autel.getMissionManager();
```
或者通过相应模块类的静态方法获取对应的接口，例如获取任务模块接口：
```
	MissionManager missonManager = AModuleMission.missionManager();
```

## 任务模块

任务模块目前支持三种任务：WaypointMission(航点任务)、OrbitMission(环绕任务)、FollowMission(跟随任务)，所有任务由任务管理器执行相关操作，任务管理器的具体操作有：prepare(准备)、start(开始)、pause(暂停)、resume(继续)，cancel(取消)、download(下载正在执行的任务)。

环绕任务为例,相关操作代码如下：

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
使用MissionManager来准备环绕任务orbitMission
```
	MissionManager myMissonManager = Autel.getMissionManager();
	myMissonManager.prepareMission(mOrbitMission, new CallbackWithOneParamProgress<Boolean>() {
                        @Override
                        public void onProgress(float v) {

                        }

                        @Override
                        public void onSuccess(Boolean aBoolean) {
                            
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                            
                        }
                    });
```
开始执行环绕任务
```
	myMissonManager.startMission(new CallbackWithNoParam() {
                        @Override
                        public void onSuccess() {
                          
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                           
                        }
                    });
```
暂停执行环绕任务
```
	myMissonManager.pauseMission(new CallbackWithNoParam() {
                        @Override
                        public void onSuccess() {
                          
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                           
                        }
                    });
```
继续执行环绕任务
```
	myMissonManager.resumeMission(new CallbackWithNoParam() {
                        @Override
                        public void onSuccess() {
                          
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                           
                        }
                    });
```
取消执行环绕任务
```
	myMissonManager.cancelMission(new CallbackWithNoParam() {
                        @Override
                        public void onSuccess() {
                          
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                           
                        }
                    });
```
下载正在执行的任务
```
	myMissonManager.downloadMission(new CallbackWithOneParamProgress<AutelMission>() {
                        @Override
                        public void onProgress(float v) {

                        }

                        @Override
                        public void onSuccess(AutelMission autelMission) {
                            Toast.makeText(applicationContext, R.string.mission_download_notify, Toast.LENGTH_SHORT).show();
                            progressBarDownload.setVisibility(View.GONE);
                        }

                        @Override
                        public void onFailure(AutelError autelError) {
                            Toast.makeText(applicationContext, autelError.getDescription(), Toast.LENGTH_LONG).show();
                            progressBarDownload.setVisibility(View.GONE);
                        }
                    });
```
**下载正在执行的任务返回的是实际执行任务的父类对象AutelMission，需要开发者自行向下转型**
```
if(autelMission instanceof OrbitMission){
	OrbitMission downloadMission = (OrbitMission)autelMission;
}
```
- 获取任务模块服务
- 操作任务
- 注意事项 
  1. 任务的生成；
  2. 任务执行状态的实时监听；
  3. 地图坐标的转换；
