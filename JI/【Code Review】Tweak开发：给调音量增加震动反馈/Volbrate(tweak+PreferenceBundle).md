
[TOC]

# Volbrate（tweak+PreferenceBundle）

最近分析一个开源越狱插件--[Volbrate][1]，这个插件是给iPhone音量键增加震动的，功能很简单。亮点在于，这个插件提供了在手机设置中做一些功能修改和设定

如图所示：    
<img src="media/15056458276103/79CFA5056A24202ABBEBE48B5BF5C87E.png" width = "300" height = "550" alt="图片名称" align=center />

通过搜索知道是利用了PreferenceBundle，theos第9个模板--preference_bundle_modern。只写过tweak，对PreferenceBundle不了解，所以就边分析边学怎么写PreferenceBundle。

## tweak

有关tweak的编写，资料好多，就不再说了。推荐看狗神的《iOS应用逆向工程》第二版

### plist文件

![plist][3]

可知，其作用于springboard

### xm文件

简单来说，就是hook音量键，然后添加震动功能。
关键hook代码：

```mm
%hook VolumeControl
	
	-(void)increaseVolume {
		if(volVibrationOptions == 2) {
			[FeedbackCall vibrateDevice];
		} if(volVibrationOptions == 1 && volMax == YES){
			[FeedbackCall vibrateDevice];
		} else {
			%orig;
		}
	}
		
	-(void)decreaseVolume {
		if(volVibrationOptions == 2) {
			[FeedbackCall vibrateDevice];
		} if(volVibrationOptions == 1 && volMin == YES){
			[FeedbackCall vibrateDevice];
		} else {
			%orig;
		}
	}
	
	-(float)volume {
		x = %orig;
		if(x == 0){
			volMin = YES;
		} if(x == 1){
			volMax = YES;
		} if(x > 0 && x < 1) {
			volMin = NO;
			volMax = NO;
		}
		
	return %orig;
	}
%end
```
由代码可知，hook VolumeControl类里面的三个函数。    
`-(float)volume`  获取音量值    
`-(void)increaseVolume`  增加一格音量    
`-(void)decreaseVolume`   减少一格音量    

### 震动实现

tweak中震动的实现是 调用一个private API ：`AudioServicesPlaySystemSoundWithVibration`    
Apple官方并没有这个函数的文档。搜索这个函数，出来最多就是[are there apis for custom vibrations in ios][4]。翻译部分信息：    
其函数原型:    
`void AudioServicesPlaySystemSoundWithVibration(SystemSoundID inSystemSoundID, id arg, NSDictionary* vibratePattern)`

- 第一个参数：inSystemSoundID类型为SystemSoundID，就像调用[AudioServicesPlaySystemSound:][5]函数一样，赋值为`kSystemSoundID_Vibrate`可以产生短暂的震动   
- 第二个参数：不重要，传参数为nil就可   
- 第三个参数：vibratePattern类型为`NSDictionary`, 根据源代码有两个key, 一个为设置震动时长与连续震动之间的间隙。一个为设置震动的强度。

```mm
+ (void)vibrateDeviceForTimeLengthIntensity:(CGFloat)timeLength vibrationIntensity:(CGFloat)vibeIntensity {

	NSMutableDictionary* dict = [NSMutableDictionary dictionary];
	NSMutableArray* arr = [NSMutableArray array];
 
	[arr addObject:[NSNumber numberWithBool: YES]]; //vibrate for time length
	[arr addObject:[NSNumber numberWithInt: timeLength*1000]];

	[arr addObject:[NSNumber numberWithBool: NO]];
	[arr addObject:[NSNumber numberWithInt: 50]];
    
	[dict setObject: arr forKey:@"VibePattern"];
	[dict setObject:[NSNumber numberWithFloat: vibeIntensity] forKey:@"Intensity"];
    
	AudioServicesPlaySystemSoundWithVibration(kSystemSoundID_Vibrate, nil, dict);

}
```

## preferenceBundle

有意思的是怎么才能写PreferenceBundle，在设置中随时修改某些参数。达到修改tweak功能的作用。比如可以做一个 是否启用hook的开关...    
Preference Bundles可以作为iPhone设置中的扩展程序，开发者能编写自己想要的bundles，安装后位于手机`/Library/PreferenceBundles/`目录下。

- Theos的PreferenceBundles模板结合了MobileSubstrate的PreferenceLoader   
PreferenceLoaders是MobileSubstrate其中的一个工具，可以把tweak扩展PreferenceBundles注入到iOS的设置中

- Tweak.xm不能直接调用PreferenceBundle来获取一些修改后的变量值，而是通过另一种方式，比如从某个plist文件读取，变量的plist文件位于`/User/Library/Preferences`目录。


### 新建PreferenceBundle工程

PreferenceBundles作为tweak的扩展，**直接就在tweak的工程目录**新建PreferenceBundles工程   
利用theos nic.pl来创建新建PreferenceBunldes工程

![][7]

如图依次填入，工程名，BundleID，工程前缀    
最后的目录结构如图：

![][8]

这时候tweak的makefile会自动增加一些东西

![][9]

### PreferenceBundle文件

- entry.plist      
确定在设置应用中的入口图标，文字等。
![][10]

- Info.plist     
这个文件保存工程信息，不需做什么修改

- Root.plist   
Root.plist可以看做是PreferenceBundle的UI布局文件。其中要跟tweak交互的变量就声明在这里，比如：

    ```xml
    <key>key</key>
    <string>varName</stirng>
    ```
    **entry.plist 和 Root.plist文件的所用的键值详细内容，可参考[Preferences specifier plist][11]**
    Info.plist和Root.plist都在Resources文件夹里。且工程所用到图片，图标文件也保存在Resources目录下。
    
- 代码文件    
模板会新建两个文件，`XXXRootListController.h` 和 `XXXRootListController.m`，`XXX`就是之前设置的工程前缀。      
`XXXRootListController`必须继承`PSListController`或者`PSViewController`，且必须实现`- (id)specifiers`方法，因为`PSListController`依赖`_specifiers`来获得metadata和group。    
[iphonedevwiki][6]的示例代码：

    ```mm
    - (id)specifiers {
    	if (!_specifiers){
    		_specifiers = [[self loadSpecifiersFromPlistName: kNameOfPreferencesPlist target: self] retain];
    	}
    	return _specifiers;
    }
    ```
    kNameOfPreferencePlist指的就是Root.plist。
    这些theos都已经替我们做好了。其他逻辑代码就写在XXXRootListController.m里，可以有多个.m文件。   
    如何在PreferenceBundle中设置和读取要交互的变量，具体方法可参考[iphonedevwiki][6]

### 加载PreferenceBundle

在tweak的constructor（%ctor）中完成PreferenceBundle的加载，
[Preferences][12]的示例代码：

```mm
@interface NSUserDefaults (Tweak_Category)
- (id)objectForKey:(NSString *)key inDomain:(NSString *)domain;
- (void)setObject:(id)value forKey:(NSString *)key inDomain:(NSString *)domain;
@end

static NSString *nsDomainString = @"com.your.tweak";
static NSString *nsNotificationString = @"com.your.tweak/preferences.changed";
static BOOL enabled;

static void notificationCallback(CFNotificationCenterRef center, void *observer, CFStringRef name, const void *object, CFDictionaryRef userInfo) {
	NSNumber *n = (NSNumber *)[[NSUserDefaults standardUserDefaults] objectForKey:@"enabled" inDomain:nsDomainString];
	enabled = (n)? [n boolValue]:YES;
}

%ctor {
	// Set variables on start up
	notificationCallback(NULL, NULL, NULL, NULL, NULL);

	// Register for 'PostNotification' notifications
	CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(),
		NULL,
		notificationCallback,
		(CFStringRef)nsNotificationString,
		NULL,
		CFNotificationSuspensionBehaviorCoalesce);

	// Add any personal initializations
}

```
`notificationCallback()`获取需要的变量值。
`CFNotificationCenterAddObserver()`是`CoreFoundation/CFNotificationCenter.h`中的函数，注册监听事件，设置监听相关变量改变后所做的事。

- 第一个参数：`CFNotificationCenterGetDarwinNotifyCenter()`，不变
- 第二个参数：NULL
- 第三个参数：监听到消息后做所的事，一般是重新获取变量值。所以填notificationCallback
- 第四个参数：Darwin消息字符串，在Root.plist文件中设置相关变量的消息字符串，然后在Tweak.xm文件中要写上对于的消息字符串。key为PostNotification。当对应的变量改变时，就会发送这个消息字符串。然后监听事件就会接受到消息。
比如在Root.plist文件中：

    ```xml
    <key>key</key>
    <string>varName</stirng>
    <key>PostNotification</key>
    <string>thisVarDarwinNotification</string>
    ```
- 第五个参数：NULL
- 第六个参数：`CFNotificationSuspensionBehaviorCoalesce`，不变

### makefile

makefile的编写跟tweak差不多

### 编译

编译是跟tweak一起编译，不用做其他操作。

# 总结

如何写PreferenceBundle的中文资料没找到，所以参考的都是英文资料，所以有些地方翻译的不好，大致意思应该明白。写的较为简单，算是基本了解怎么写PreferenceBundle吧。

# 参考链接

https://github.com/derv82/Exchangent/wiki/Part-6:-Preferences,-Preferences,-a-little-Tweak,-and-Heaps-of-More-Preferences
http://sharedinstance.net/2015/02/settings-the-right-way-redux/
http://iphonedevwiki.net/index.php/PreferenceBundles
https://developer.apple.com/documentation/audiotoolbox/ksystemsoundid_vibrate
http://sharedinstance.net/2015/02/settings-the-right-way-redux/


[1]:https://github.com/LacertosusRepo/Open-Source-Tweaks/tree/master/Volbrate
[2]:media/15056458276103/79CFA5056A24202ABBEBE48B5BF5C87E.png
[3]:media/15056458276103/15056551271228.jpg
[4]:https://stackoverflow.com/questions/12966467/are-there-apis-for-custom-vibrations-in-ios
[5]:https://developer.apple.com/documentation/audiotoolbox/ksystemsoundid_vibrate
[6]:http://iphonedevwiki.net/index.php/PreferenceBundles
[7]:media/15056458276103/15063392531747.jpg
[8]:media/15056458276103/15064108654004.jpg
[9]:media/15056458276103/15064988791579.jpg
[10]:media/15056458276103/15064999858045.jpg
[11]:http://iphonedevwiki.net/index.php/Preferences_specifier_plist
[12]:http://iphonedevwiki.net/index.php/Preferences

