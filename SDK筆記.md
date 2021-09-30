
1. Adjust 归因
nstall Referrer 是一种唯一标识符，可用来将应用安装归因至来源。Adjust SDK 需要该信息进行归因
compileOnly 'com.android.installreferrer:installreferrer:2.2'
2. Unity 的Gradle版本为5.1.1 需要更换为新的gradle版本
 更换6.9----JDK11 成功
 替换Gradle6.8在Unity各版本中
 JDK11直接在gradleTemple.properties中添加org.gradle.java.home=C:/Program Files/Java/jdk-11
3. Crashlytics 中的调试
adb shell setprop log.tag.FirebaseCrashlytics DEBUG
adb logcat -s FirebaseCrashlytics
4. Performance Monitoring 调试
adb logcat -s FirebasePerformance
5. RemoteConfig 需要翻墙才能初始化成功
