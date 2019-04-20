# 前言
公司项目近期正在将 XML 布局文件转换为纯代码编写，但是由于之前为了避免 65535 问题和开发环境编译速度，所以 `build.gradle` 中配置了一个 `minSdkVersion` 为 21 的 `productFlavors`。这就导致在转换工程中出现了很多调用高版本方法(比如 `View#setElevation`)的问题，Lint 也不会提示开发者修改。		
为了避免这个问题，开始寻找解决的办法，经过搜索后发现没有类似的问题，这里记录一下我的方法，希望能给有类似问题的朋友一点帮助。由于个人水平有限，有错误请指出。

## 思路
1. 找到能够设置 Lint 检查 minSdkVersion 的方法；	
很遗憾，没能找到，有的话也就没有此文章了。	
不过我已经向 Google 提出了 [Feature Request](https://issuetracker.google.com/issues/130791559)，也不知道会不会被采纳。

2. 查看 AGP(Android Gradle Plugin) 和 Lint 源码， 找到关键步骤，通过 Gradle 插入 Task 调用 `setMinSdkVersion()`。	
很遗憾，由个人水平有限，而且这个方法花费时间较多，所以耗费一段时间后就放弃了。这里贴出一下相应源码解析:
[Android Lint工作原理剖析](http://www.androidchina.net/5106.html)  
[
从Android Plugin源码开始彻底理解gradle构建：初识AndroidDSL（一)](https://blog.csdn.net/verymrq/article/details/80358111)	
[Android Gradle Plugin源码分析](https://www.jianshu.com/p/11f030b2034f)

3. 自定义 Lint Rules，复制 Api 相应的源码，更改源码中的 `minSdkVersion`，偷梁换柱。	
可行性高，本文后续就讲解该方案的实现过程。

## 自定义 Lint Rules，偷梁换柱
这里偷个懒，如何自定义 Lint 请查看下列文章(别人写得好，也更详细)：	
[【我的Android进阶之旅】Android自定义Lint实践](https://blog.csdn.net/ouyang_peng/article/details/80374867)	
[美团外卖Android Lint代码检查实践](https://tech.meituan.com/2018/04/13/waimai-android-lint.html)	

####  注意: 为了能够让自定义 lint 能够在编码阶段实时检查，请将根目录下的 AGP 版本与 Android Studio 版本保持一致，否则可能不会生效！

由于上述文章都是以 AGP 2.X 版本为背景进行开发的，但是 AGP 应该都是 3.X 版本了，所以这里主要讲一下其中的区别：
1. 引入自定义 `lint.jar` 不再需要采用 [LinkedIn](https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle) 的方案，官方已提高 `lintChecks` 支持。 	
代码请参考 [googlesamples/android-custom-lint-rules/android-studio-3](https://github.com/googlesamples/android-custom-lint-rules/tree/master/android-studio-3)。	
> 在 AGP 3.5.0 中，我还发现了 `lintPublish` 这个与 lint 相关的关键字，没发现与 `lintChecks` 的区别。目前发现的作用是，如果要像 AGP 2.X 版本将 lint.jar 打包进 aar 的话，使用 `lintChecks` 是不行的，`lintPublish` 才会生效。
有兴趣的朋友可以 [AGP 3.5.0-alpha10](https://mvnrepository.com/artifact/com.android.tools.build/gradle/3.5.0-alpha10) 下载 jar 包查看对应源码, 位置是 `com.android.build.gradle.internal.TaskManager#createCustomLintChecksConfig(Project)`。	

2. 自定义 lint 的 java 工程中的 manifest 配置参数有所改变。
	jar {
        // 向 java 中的 manifest 写入
        manifest {
            // 指定自定义的 Lint 检查类
			// 为了保险起见，其实两个都可以加上。
            // AGP 3.X
            attributes("Lint-Registry-v2": "com.example.lint.MyIssueRegistry")

            // AGP 2.X
            attributes("Lint-Registry": "com.example.lint.MyIssueRegistry")
            }
       }
    
  
 
通过查看 `lint-checks` 源码，可以从 `BuiltinIssueRegistry` 找到负责 Api 相关检查的类  `ApiDetector`。

但是通过 gradle 依赖的 `lint-checks` 中没有提供 .java 源码。所以需要去 [googlesource](https://android.googlesource.com/platform/tools/base/+/refs/tags/studio-3.2.1/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks) 找 java 源码，然后复制相关的源文件即可。
	
    /**
     * Looks for usages of APIs that are not supported in all the versions targeted by this application
     * (according to its minimum API requirement in the manifest).
     */
    public class ApiDetector extends ResourceXmlDetector
            implements SourceCodeScanner, ResourceFolderScanner {
        public static final AndroidxName REQUIRES_API_ANNOTATION =
                AndroidxName.of(SUPPORT_ANNOTATIONS_PREFIX, "RequiresApi");
        public static final String SDK_SUPPRESS_ANNOTATION = "android.support.test.filters.SdkSuppress";
        /**
         * Accessing an unsupported API
         */
        @SuppressWarnings("unchecked")
        public static final Issue UNSUPPORTED =
                Issue.create(
                        "NewApi_Mock", // ①
                        "Calling new methods on older versions",
                        "This check scans through all the Android API calls in the application and "
                                + "warns about any calls that are not available on **all** versions targeted "
                                + "by this application (according to its minimum SDK attribute in the manifest).\n"
                                + "\n"
                                + "If you really want to use this API and don't need to support older devices just "
                                + "set the `minSdkVersion` in your `build.gradle` or `AndroidManifest.xml` files.\n"
                                + "\n"
                                + "If your code is **deliberately** accessing newer APIs, and you have ensured "
                                + "(e.g. with conditional execution) that this code will only ever be called on a "
                                + "supported platform, then you can annotate your class or method with the "
                                + "`@TargetApi` annotation specifying the local minimum SDK to apply, such as "
                                + "`@TargetApi(11)`, such that this check considers 11 rather than your manifest "
                                + "file's minimum SDK as the required API level.\n"
                                + "\n"
                                + "If you are deliberately setting `android:` attributes in style definitions, "
                                + "make sure you place this in a `values-v`*NN* folder in order to avoid running "
                                + "into runtime conflicts on certain devices where manufacturers have added "
                                + "custom attributes whose ids conflict with the new ones on later platforms.\n"
                                + "\n"
                                + "Similarly, you can use tools:targetApi=\"11\" in an XML file to indicate that "
                                + "the element will only be inflated in an adequate context.",
                        Category.CORRECTNESS,
                        6,
                        Severity.ERROR,
                        new Implementation(
                                ApiDetector.class,
                                EnumSet.of(Scope.JAVA_FILE, Scope.RESOURCE_FILE, Scope.MANIFEST),
                                Scope.JAVA_FILE_SCOPE,
                                Scope.RESOURCE_FILE_SCOPE,
                                Scope.MANIFEST_SCOPE));
        /**
         * Accessing an inlined API on older platforms
         */
        public static final Issue INLINED = //...省略
        /**
         * Method conflicts with new inherited method
         */
        public static final Issue OVERRIDE = //...省略
        /**
         * Attribute unused on older versions
         */
        public static final Issue UNUSED = //...省略
        /**
         * Obsolete SDK_INT version check
         */
        public static final Issue OBSOLETE_SDK = //...省略
    } 

1. ① 处，是为了防止与自带 lint 中的 `NewApi` 区别。
2. 创建一个常量 `private static final AndroidVersion MIN_SDK_VERSION = new AndroidVersion(15, (String)null);` ，然后搜索 `ApiDetector` 中 `minSdkVersion` 变量，将 `MIN_SDK_VERSION` 赋值给 `minSdkVersion`，这样检查时获取的 minSdkVersion 就是固定的 15 了，当然这里的 15 你可以任意修改。
3. 在 `VersionChecks.java` 中，内部类 `ApiCheckGraph` 会继承 `ControlFlowGraph`，但是 `ControlFlowGraph` 在 `lint-api` 中不存在，后面发现只有一个方法使用此类，而这个方法也没有调用，遂注释之，以成功编译。

最后按照前面提到的自定义 lint 方式编译即可。

参考资料：

[Android Lint工作原理剖析](http://www.androidchina.net/5106.html)  
[
从Android Plugin源码开始彻底理解gradle构建：初识AndroidDSL（一)](https://blog.csdn.net/verymrq/article/details/80358111)	
[Android Gradle Plugin源码分析](https://www.jianshu.com/p/11f030b2034f)
[【我的Android进阶之旅】Android自定义Lint实践](https://blog.csdn.net/ouyang_peng/article/details/80374867)	
[美团外卖Android Lint代码检查实践](https://tech.meituan.com/2018/04/13/waimai-android-lint.html)	
[googlesamples/android-custom-lint-rules/android-studio-3](https://github.com/googlesamples/android-custom-lint-rules/tree/master/android-studio-3)
[googlesource](https://android.googlesource.com/platform/tools/base/+/refs/tags/studio-3.2.1/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks)	
[Writing Custom Lint Rules](http://tools.android.com/tips/lint-custom-rules)	
[https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle](https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle)	
