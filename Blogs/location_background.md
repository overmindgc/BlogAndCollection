# 如何通过开启定位能力让你的iOS App常住后台并支持kill后自动唤醒

### 背景
有一次老板提出了这种需求：我们的监控App必须能一直在后台运行并且杀死了也能自动重启！然后泪流满面的iOS开发只能默默的寻找解决办法。。。别说，还真有！
### iOS系统常驻后台的几种办法
在XCode项目的Capabilities选项中，有这么几种后台运行模式的权限：

![Capabilities的后台模式选项](http://upload-images.jianshu.io/upload_images/671658-cf466abe17ae7c61.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
大概就是音频、定位、蓝牙、远程通知等等，还有具有voip能力的应用可以通过在系统注册socket连接来实现后台唤醒。
> Note：需要注意的是这些能力必须匹配App的功能，如果App中没有合理的使用理由的话上架会被拒绝的。

因为开发的是一款监控位置的软件，所以我在这里只写一下后台位置权限的使用。
### 实现方法
##### 1、勾选Location updates
首先需要吧Capabilities配置里的Location updates选项勾上，不然是没有定位权限的。
##### 2、添加CoreLocation.framework
定位功能是CoreLocation里的，需要先引入这个framework
##### 3、在Info.plist里添加定位权限描述文字
需要添加这2条描述文字：
`Privacy - Location Always Usage Description - 持续的后台定位`
`Privacy - Location When In Use Usage Description - 使用中定位`
iOS11还需要再添加这一条
`Privacy - Location Always and When In Use Usage Description - 定位权限选择说明`
> Note：这里在iOS11有了变化，如果要使用始终定位权限，强制要求也必须给用户选择使用中定位的选项。
如果选择了使用中定位的权限，App如果在后台继续定位就会在顶部显示正在使用定位的蓝条，这样就避免了App偷偷的在后台定位。

##### 4、创建并配置CLLocationManager
CLLocationManager的各项参数配置就不详细说了，根据需求设置即可。重点说一下启动持续定位的方法。
##### 5、开启持续精确定位
iOS9以上需要在开启定位前调用一下这两个方法（9以下只需要调用第二个），用户就会弹出定位权限申请
`[locationManager setAllowsBackgroundLocationUpdates:YES];`
`[locationManager requestAlwaysAuthorization]`
然后开启定位时调用下面的方法就可以在didUpdateLocations接受定位信息了
`[locationManager  startUpdatingLocation]`
> Tips：检测定位权限开启可以在didFailWithError回调中接收到响应，返回的error的code是kCLErrorDenied类型的，就是未开启定位权限。
低电量模式有时候可能会影响到后台定位，可以通过
`[[NSProcessInfo processInfo] isLowPowerModeEnabled] (iOS9以上)`来判断。

经过实际使用，基本上开启持续精确定位以后，后台可以一直存活。
不过到夜里会有比较大的概率被系统kill掉。
重度使用其他软件导致系统资源紧张，也可能会被系统kill掉。
还有如果未开启4G网络通话功能打电话或者关闭4G信号并在室内导致定位失败时间较长，也可能会被kill掉。
用户主动在多任务进程里关掉App，也会停止定位。

#### 重点来了，如何支持被kill掉以后能够后台自动重启
CLLocationManager还有一个持续定位的方法
`startMonitoringSignificantLocationChanges`
这个是粗略定位，只有在4G基站切换的时候才会响应didUpdateLocations回调。本来这个是用在不需要那么精确定位的场景下省电开启的。
那这个方法有什么特殊的呢，那就是支持后台自动唤醒！而且可以和`startUpdatingLocation`同时开启！
> Note：这个基站定位只有在移动超过一定距离的时候，才会有响应，大概最少也得1公里的样子，所以也不是实时就能唤醒的，但是通过定位能力也只能做到这里了😂
##### 使用方法：
在调用`[locationManager  startUpdatingLocation]`的时候，也调用一下
`[locationManager  startMonitoringSignificantLocationChanges]`
然后只需要在`AppDelegate`里的
`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`
回调中加入下边的代码，就可以了。
````
/*********应用kill以后基站定位唤醒*************/
if ([launchOptions objectForKey:UIApplicationLaunchOptionsLocationKey]) {
    NSLog(@"在后台被唤醒");
    // do something，这里就可以再次调用startUpdatingLocation，开启精确定位啦
}
````
当然了，用户必须选择了始终定位权限，才可能从后台自动启动，这也是为了保护用户隐私权和节省电力，不想被偷偷监控的，把所有软件定位权限都设置成使用时就可以了，因为iOS11强制App必须有这个选项，所以抓紧升级iOS11吧 😆。