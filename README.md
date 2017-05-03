= 集成Autel SDK 开发环境
* gradle引入SDK资源包
在项目gradle文件中配置maven服务器路径
```
allprojects {
    repositories {
        maven(){
            url "http://10.240.12.2:8081/artifactory/autel-snapshot"
        }
    }
}
```
* 初始化SDK
在需要使用SDK功能前，初始化相关SDK功能，例如在自定义Application中初始化SDK服务：
```
public class TestApplication extends Application {
    public void onCreate() {
        super.onCreate();
        String appKey = "<SDK license should be input>";
        Autel.init(this, appKey, new CallbackWithNoParam() {
            @Override
            public void onSuccess() {
            }

            @Override
            public void onFailure(AutelError error) {
            }
        });
    }
}
```

(NOTE)appKey 为开发者在Autel开发平台申请的应用关联钥匙

* 获取各个模块相关服务
* 销毁SDK服务

= 任务模块调用示例
- 获取任务模块服务
- 操作任务
- 注意事项 
  1. 任务的生成；
  2. 任务执行状态的实时监听；
  3. 地图坐标的转换；
