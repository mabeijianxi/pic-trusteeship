	说明：本篇不撸代码，只玩编译,其包含了Android studio 2.2最新的JNI玩法
	编译环境：macOS 10.12.3
	工具包含：Android Studio 2.2  NDK-r14 
 在Android下要玩jni首先下载ndk是必须的，可以直接去<a href='https://developer.android.google.cn/ndk/downloads/index.html'>https://developer.android.google.cn/ndk/downloads/index.html</a>下载，当然我们家AS为开发者也提供了便捷<br><br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_1.png' /><br><br>只需如图勾选然后OK即可，我的版本是r14，值得一提的是 **google ndk-build** 命令在  **r13** 后默认使用  **Clang**，并将在后续版本中移除  **GCC**，其编译速度更快、编译产出更小、出错提示更友好。

## 一、徒手编写Android.mk然后ndk-build编译：
这种编译其实是用make工具来玩的，在  linux 徒手写并编译过c的应该很清楚，通过编写makefile，然后再用make编译已经比不停的用gcc命令逐个编译要爽很多,但是 makefile 的编写还是有点蛋疼。程序员都是化繁为简善解人意的，通过 ndk 工具我们无需自己写 makefile 了，现在你只要安心撸自己关心的代码就行了。<br><br>
1、在main下新建 **jni** 目录，如图：<br><br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_2.png' /><br><br>
2、再新建一个  **c**  或者  **c++** 文件，如图：<br><br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_3.png' /><br><br>
3、在java里面声明个  **native** 方法：<br>
<pre><code>
	    private native String jniTellMeWhy(String hiJni);
</code></pre>
4、copy全类名<br><br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_4.png' /><br><br>然后去我们新建的那个  **hi_jni.cpp**  里面去声明一个方法，这里就添加头文件了，直接干。命名规则是死的，粘贴一下把"."换成"  **_**  "再加上“  **_方法名**  ”,我们是有返回值的且是个  **String**  的，对应的就是  **jstring**   ，最前面拼上固定的“  **Java_**  ”。我们传入了一个参数，但是规定是每个函数默认都会两个参数，一个是  **JNIEnv**  指针类型的结构体，一个是调用者对象，比如我们这里就是MainActivity对象，其实玩过  **c++**  的都知道里面每个函数其实默认也会传入个  **this**  指针的，不然一个类可以有那么多对象怎么知道是哪个对象调用的？言归正传，还有一个参数就是我们传入的  **String**  值了。如下：<br>
<pre><code>
	#include <jni.h>
	#include <stdio.h>
	#include <stdlib.h>
extern "C"
jstring
Java_com_example_jianxi_jnidemo_MainActivity_jniTellMeWhy(JNIEnv *env, jobject obj, jstring str) {
    const char *question = env->GetStringUTFChars(str, JNI_FALSE);
    char *answer = "fuck,no why!!!";
    char *data = (char *) malloc(strlen(question) + strlen(answer)+1);
    strcpy(data,question);
    strcat(data, "JNI说:");
    strcat(data, answer);
    return env->NewStringUTF(data);
}
</code></pre>
我这里用的是  **c++**  所以得加上 **extern "C"**   ，原因很简单，在  **C++**  中函数在编译的时候会拼接上参数，这也是  **c++**  中函数重载的处理机制，比如一个  **set(int a)**   和一个  **set(int a,int b)**  ，在编译的时候就变成了  **set_int**  与  **set_int_int**  ，我们加上  **extern ”C“**   就表示大爷想按照C来编译，所以函数名字后面就不会拼接上参数类型了。<br><br>
5、在jni目录下新建两个文件一个叫  **Android.mk**  ,一个叫  **Application.mk**  。<br><br>
6、编写Android.mk，最简单的编写如下，后面将介绍一些稍微牛逼点点的。<br>
<pre><code>
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := hi_jni
LOCAL_SRC_FILES := hi_jni.cpp
LOCAL_C_INCLUDES += $(LOCAL_PATH)
 #LOCAL_LDLIBS := -llog
include $(BUILD_SHARED_LIBRARY)
</code></pre>
 **LOCAL_PATH**  ：是得最先配置的，它用于在开发tree中查找源文件。<br><br>
 **include $(CLEAR_VARS)**  ： **CLEAR_VARS**  变量指向特殊  **GNU Makefile**  ，可为您清除许多  **LOCAL_XXX**  变量，例如  **LOCAL_MODULE**   、  **LOCAL_SRC_FILES**  和   **LOCAL_STATIC_LIBRARIES**  。 请注意，它不会清除。<br><br>
 **LOCAL_PATH**  :此变量必须保留其值，因为系统在单一  **GNU Make**  执行环境（其中所有变量都是全局的）中解析所有构建控制文件。 在描述每个模块之前，必须声明（重新声明）此变量。<br><br>
  **LOCAL_MODULE**  ：存储您要构建的模块的名称,并指定的想生成的  **so**  叫什么名字。当然生成产物的时候前面会自动拼接上  **lib**  ,后面会自动拼接上  **.so**  。<br><br>
  **LOCAL_SRC_FILES**  :要编译的源文件，多个文件以空格分开即可。当导入  **.a**  或者  **.so**  文件的时候一个模块只能添加一个文件，后面将演示。<br><br>
  **LOCAL_C_INCLUDES**  :可以使用此可选变量指定相对于  **NDK root**  目录的路径列表，以便在编译所有源文件（C、C++ 和 Assembly）时添加到 include 搜索路径，通常是原文件地址、头文件地址等。<br>
  **LOCAL_LDLIBS**  :这里是添加一个本地依赖库，比如可以添加一个  **log**  库，当然我没用到就注释了。<br><br>
**include $(BUILD_SHARED_LIBRARY)**  :这一行帮助系统将所有内容连接到一起， **BUILD_SHARED_LIBRARY**  变量指向  **GNU Makefile**  脚本，用于收集您自最近 include 后在   **LOCAL_XXX**  变量中定义的所有信息。 此脚本确定要构建的内容及其操作方法。 **BUILD_SHARED_LIBRARY**  代表动态库， **BUILD_STATIC_LIBRARY** 代表静态库 。<br><br>


7、编写  **Application.mk**  ：
<pre><code>
	# 指定生成哪些cpu架构的库
	APP_ABI := armeabi-v7a
	# 此变量包含目标 Android 平台的名称
	APP_PLATFORM := android-22
</code></pre>

8、在  **jni**  目录下面打开命令行工具，然后执行  **ndk-build**  ,即可在  **libs**  目录下得到产物:<br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_7.png' /><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_8.png' /><br><br>
9、把产物放到  **jniLibs**  下面（当然你可以在采用  **builde.gradle**  的  **sourceSets**  里面改变其路径， **jniLibs.srcDirs=['src/main/libs']）**  。<br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_9.png' /><br><br>

10、  **Java**  层调用：
<pre><code>
 private  String TAG= getClass().getSimpleName();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        String s = jniTellMeWhy("我是MainActivity,tell me why!");
        Log.e(TAG,s);
    }
    private native String jniTellMeWhy(String hiJni);
    static {
        System.loadLibrary("hi_jni");
    }
</code></pre>
结果是：<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_5.png' /><br><br>
  **添加一预构建库编译：**  <br><br>
其实和咋们Android中的添加依赖差不多，我们要编译一个原生库，这个库的功能是可以按照  **H264**  编码视频，然后我们不想自己写那么多代码，所以我们引入开源的  **libx264**  ，我们拿到编译好的  **libx264.a**  或者  **libx264.so**  和其头文件，这个时候我们只需要导入一起编译即可：
目录结构如图<br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_10.png' /><br><br>
有了头文件我们即可include入我们的  **hi_jni.cpp**  里面自由蹂蹑。<br><br>
接下来修改下  **Android.mk**  。<br>
<pre><code>
LOCAL_PATH := $(call my-dir)
	#第一组
include $(CLEAR_VARS)
LOCAL_MODULE := x264
LOCAL_SRC_FILES := $(LOCAL_PATH)/x264/libx264.a
LOCAL_C_INCLUDES += $(LOCAL_PATH)/include/
include $(PREBUILT_STATIC_LIBRARY)
	#第二组
include $(CLEAR_VARS)
LOCAL_MODULE := hi_jni
LOCAL_SRC_FILES := hi_jni.cpp
LOCAL_C_INCLUDES += $(LOCAL_PATH)
LOCAL_STATIC_LIBRARIES := x264
include $(BUILD_SHARED_LIBRARY
</code></pre>
如果我们导入  **.a**  的静态库的话第一组就如上所写，没添加一组的时候必须执行  **include $(CLEAR_VARS)**  ，  **LOCAL_MODULE**  的值就各自喜好了，第一组的  **LOCAL_SRC_FILES**  我们指向想导入的静态库地址，第一组的  **LOCAL_C_INCLUDES**  指向其头文件地址，然后  **include $(PREBUILT_STATIC_LIBRARY）**  代表生成静态预构建。
我们在第二组中引用第一组的静态预构建也就是  **LOCAL_STATIC_LIBRARIES := x264**   ，引用动态预构建只需把  **STATIC**  修改为  **SHARED**  即可。配置完成即可在当前目录打开命令行执行  **ndk-build**   命令生成产物。如果第一组你指定的是  **.so**  的动态库，使用的时候也得在  **java**  层  **System.loadLibrary("x264")**  。
## 二、通过配置AS中build.gradle来编译：<br>
	这种方式比上一中又简化了很多，无需再自己编写  **Android.mk**  了，但原理都是一样的。
1、在  **main**  下新建  **jni**  目录，如图：<br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_2.png' /><br><br>
2、再新建一个  **c**  或者  **c++**  文件，如图：<br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_3.png' /><br><br>
3、找到你项目的  **gradle.properties**   ，添加一行  **android.useDeprecatedNdk=true**    <br><br>
4、打开你主  **Module**  的  **build.gradle**  ,在  **defaultConfig**  里添加:<br><br>
<pre><code>
	ndk{
            moduleName 'hi_jni'
            abiFilter 'armeabi-v7a'
        }
</code></pre>
名如其实， **moduleName**  是你给生成的  **So** 的取的名字，当然它会在前面拼接上 “  **lib**  ”，会在后面拼接上  **.so**  ，于是生成的名字就  **libhi_jni.so**  ,  **abiFilter**  嘛就是你想保留哪些架构类型的  **so**  ，一般  **arm**  的就够玩了，当然除了这些还有很多可配参数,比如想添加个日志库来玩玩，那就添加这行呗:  **IdLibs “log“**   。<br><br>
5、同 徒手编写Android.mk然后ndk-build编译 中的 3。<br><br>
6、同 徒手编写Android.mk然后ndk-build编译 中的 4.<br> <br>
7、同 徒手编写Android.mk然后ndk-build编译 中的 10.<br><br>

这种方式不仅 so easy,当配置好后编译器还能帮我们像在写  **Java**  代码一样做代码提示。我们可以打开 主  **Mudoul下debug->ndk**  文件夹:<br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_6.png' /><br>看自动给我们生成了  **Android.mk**  ，所以说原理都是一样的。<br><br>
## 三、通过AS的cMake插件编译：

没错，今天的重头戏就是这个，也是必须把AS升级到2.2以后才能玩的功能，有了它写原生代码可谓是如鱼得水，做到真走的徒手一千行，没有比友基！<br>

怎么玩？点点啊，我靠只需要在新建项目的时候打个钩：<br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_11.png' />,没错勾上 Incude C++ Support,然后呢？然后就是直接运行即可。<br><br>
这是自动生成的部分  **c++**  代码:<br>
<pre><code>
	extern "C"
jstring
Java_com_example_jianxi_x264test1_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */,jstring str) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
</code></pre>
这是自动生成的部分Java代码：<br>
<pre><code>

		public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
}
</code></pre>
  **自动生成的build.gradle里面多出了这两个东西**  <br><br>
  **android**  里多了：
<pre><code>
	externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
</code></pre><br>
  **defaultConfig**  里多了:
<pre><code>
	externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }
</code></pre>
在主  **Mudule**  的跟目录下多了个  **CMakeLists.txt**  ，我们定制自己的原生代码的时候主要就是修改  **CMakeLists.txt**  里面的配置，下面也会详细讲解.<br><br>
### 在原生代码里面添加本地库：
这里以  **log**  库为例， **log**  库是  **android**  下的，如果我们新建项目的时候勾选上了    **Incude C++ Support**  ，那么自动生成的  **CMakeLists.txt**  里面默认会为我们添加  **log**  库，置于  **CMakeLists.txt**  的配置一会儿道来。<br><br>

下面是导入  **log**  库的头文件，并且宏定义  **log**  打印函数的代码:
<pre><code>
	#include <android/log.h>
	#define LOG_TAG "来自jni:"
	#define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)
</code></pre>
我们这里宏定义了个  **LOG_TAG** ,并宏定义打印函数  **__android_log_print**  ，我们传入   **ANDROID_LOG_ERROR**  ，所以是E级别。<br><br>
使用:<br>
<pre><code>
extern "C"
jstring
Java_com_example_jianxi_jnidemo_MainActivity_jniTellMeWhy(JNIEnv *env, jobject obj, jstring str) {
    const char *question = env->GetStringUTFChars(str, JNI_FALSE);
    char *answer = "fuck,no why!!!";
    char *data = (char *) malloc(strlen(question) + strlen(answer)+1);
    strcpy(data,question);
    strcat(data, "JNI说:");
    strcat(data, answer);
    LOGE("我在hi_jni.cpp里面和你撩呢!");
    return env->NewStringUTF(data);
}
</code></pre>
  **输出结果:**  <br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_12.png' /><br><br>
当然我们平时玩  **log**  总是有一个总开关的，这里我们也可以定义一个，这个开关我们放在  **build.gradle**  里面:<br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_13.png' /><br><br>,没错  **-D**  命令就是宏定义，这里我们宏定义了一个  **Debug**  。接下来我们在原生代码里面我们就可以根据是否定义了这个宏来决定是否输出日志。代码如下:
<pre><code>
 #include <jni.h>
 #include <stdio.h>
 #include <stdlib.h>
 # ifdef Debug
 #include <android/log.h>
 #define LOG_TAG "来自jni:"
 #define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)
 # endif
extern "C"
jstring
Java_com_example_jianxi_jnidemo_MainActivity_jniTellMeWhy(JNIEnv *env, jobject obj, jstring str) {
    const char *question = env->GetStringUTFChars(str, JNI_FALSE);
    char *answer = "fuck,no why!!!";
    
   char *data = (char *) malloc(strlen(question) + strlen(answer)+1);
   strcpy(data,question);
   strcat(data, "JNI说:");
   strcat(data, answer);
 #ifdef Debug
    LOGE("我在hi_jni.cpp里面和你撩呢!");
 # endif
    return env->NewStringUTF(data);
}
</code></pre>
### 接下我们仔细研究下 CMakeLists.txt<br>
我复制了里面自动生成的一些关键脚本：
<pre>
	add_library( # Sets the name of the library.
             hi_jni
             # Sets the library as a shared library.
             SHARED
             # Provides a relative path to your source file(s).
             # Associated headers in the same location as their source
             # file are automatically included.
             src/main/cpp/hi_jni.cpp )
 # Searches for a specified prebuilt library and stores the path as a
 # variable. Because system libraries are included in the search path by
 # default, you only need to specify the name of the public NDK library
 # you want to add. CMake verifies that the library exists before
 # completing its build.

find_library( # Sets the name of the path variable.
              log-lib
              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

 # Specifies libraries CMake should link to your target library. You
 # can link multiple libraries, such as libraries you define in the
 # build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
                       hi_jni
                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
</pre>

  **add_library**  :我们需要在里面指定三个东西，首先给  **lib**  取一个名字，然后指定作为动态库还是静态库，最后指定源文件或者库。使用  **add_library()**  向您的  **CMake**  构建脚本添加源文件或库时， **Android Studio**  还会在您同步项目后在  **Project**  视图下显示关联的标头文件。不过，为了确保  **CMake**  可以在编译时定位您的标头文件，您需要将  **include_directories()**  命令添加到  **CMake**  构建脚本中并指定标头的路径：
<pre>
	add_library(...)

   # Specifies a path to native header files.
include_directories(src/main/cpp/include/)
</pre>
   **find_library**  :将  **find_library()** 命令添加到您的  **CMake**  构建脚本中以定位  **NDK**  库，并将其路径存储为一个变量。您可以使用此变量在构建脚本的其他部分引用  **NDK**  库。<br>
  **target_link_libraries**  :指定要关联到原生库的库，第一个自然是我们  **add_library**  里面指定的库名字  **hi_jni**  库，然后可以看到  **${log-lib}**  ，也就是引用了  **find_library**  里面定义的日志库。<br>
经过上面的脚本，基本可以玩起来了。
### 引入其他编译好的静态库或者动态库
我们需要在  **add_library**  里面把我们的库  **add**  进去，也是三个参数，指定库名字，指定库类型，第三个指定为  **IMPORTED**  关键字即可，然后我们需要再添加个  **set_target_properties()**  命令，里面 也是三个参数，要为其设置属性的库名称，指定其预构建类型，如 PROPERTIES   **IMPORTED_LOCATION**  ，最后指定  **.a .so**  库的绝对地址。<br><br>
如图是我的目录结构:<br><br><img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_14.png' /><br><br>
<pre>
	add_library( # Sets the name of the library.
             hi_jni
             # Sets the library as a shared library.
             SHARED
             # Provides a relative path to your source file(s).
             # Associated headers in the same location as their source
             # file are automatically included.
             src/main/cpp/hi_jni.cpp )
 # Searches for a specified prebuilt library and stores the path as a
 # variable. Because system libraries are included in the search path by
 # default, you only need to specify the name of the public NDK library
 # you want to add. CMake verifies that the library exists before
 # completing its build.

add_library(
            x264
            STATIC
            IMPORTED
            )
set_target_properties(
    x264
    PROPERTIES IMPORTED_LOCATION
    /Users/jianxi/android/projects/o2o_2017_3_24_2.6.0/JNIDemo/app/src/main/cpp/x264/libx264.a
    )


include_directories(
    src/mian/cpp/include
)
find_library( # Sets the name of the path variable.
              log-lib
              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

 # Specifies libraries CMake should link to your target library. You
 # can link multiple libraries, such as libraries you define in the
 # build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
                       hi_jni
                       x264
                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
</pre>
由于我导入的  **x264**  静态库是  **armeabi-v7a**  架构的所以只能编译这个架构的，我需要在  **builde.gradle**  里面添加过滤:<pre>ndk{
            abiFilters "armeabi-v7a"
        }</pre>
配置完成了我们现在试试导入  **x264**  的头文件，调用下其默认参数配置函数玩玩，代码如下:
<pre><code>
 #include <jni.h>
 #include <stdio.h>
 #include <stdlib.h>
 # ifdef Debug
 #include <android/log.h>
 #define LOG_TAG "来自jni:"
 #define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)
 # endif

 #include "include/x264.h"
 extern "C"
 jstring
 Java_com_example_jianxi_jnidemo_MainActivity_jniTellMeWhy(JNIEnv *env, jobject obj,   jstring str) {
    const char *question = env->GetStringUTFChars(str, JNI_FALSE);
    char *answer = "fuck,no why!!!";
    char *data = (char *) malloc(strlen(question) + strlen(answer)+1);
    strcpy(data,question);
    strcat(data, "JNI说:");
    strcat(data, answer);
   x264_param_t * param=(x264_param_t *)malloc(sizeof(x264_param_t));
    x264_param_default(param);
 #ifdef Debug
    LOGE("我在hi_jni.cpp里面和你撩呢!");
    LOGE("i_bframe=%d\n",param->i_bframe);
    LOGE("i_level_idc=%d\n",param->i_level_idc);
 # endif
    delete param;
    return env->NewStringUTF(data);
}
</code></pre>

运行后输出结果如图:<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/pic_15.png' /><br><br>
上面演示的是添加外部静态库，添加动态  **so**  库也大同小异，把  **add_library（）**  里面的  **STATIC**  改成  **SHARED**  即可编译。<br><br>
编译就是怎么简单,我们的全部配置其实都写入了另外一个脚本文件  **cmake_build_command.txt**  它的位置如图：<br><br>
<img src='https://github.com/mabeijianxi/pic-trusteeship/blob/master/pic/jni/jni_16.png' /><br><br>
我们点开看看:<br>
<pre>
	Executable : /Users/jianxi/android/sdk/cmake/3.6.3155560/bin/cmake
arguments : 
-H/Users/jianxi/android/projects/o2o_2017_3_24_2.6.0/x264test1/app
-B/Users/jianxi/android/projects/o2o_2017_3_24_2.6.0/x264test1/app/.externalNativeBuild/cmake/debug/armeabi-v7a
-GAndroid Gradle - Ninja
-DANDROID_ABI=armeabi-v7a
-DANDROID_NDK=/Users/jianxi/android/sdk/ndk-bundle
-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=/Users/jianxi/android/projects/o2o_2017_3_24_2.6.0/x264test1/app/build/intermediates/cmake/debug/obj/armeabi-v7a
-DCMAKE_BUILD_TYPE=Debug
-DCMAKE_MAKE_PROGRAM=/Users/jianxi/android/sdk/cmake/3.6.3155560/bin/ninja
-DCMAKE_TOOLCHAIN_FILE=/Users/jianxi/android/sdk/ndk-bundle/build/cmake/android.toolchain.cmake
-DANDROID_NATIVE_API_LEVEL=14
-DCMAKE_CXX_FLAGS=
jvmArgs : 
</pre>
参数的含义看官网的吧<a href='https://developer.android.google.cn/ndk/guides/cmake.html'></a>, 撸了七七四十九个小时，实在撸不动了，这些参数可以在build.gradle里面添加,如下:
<pre>defaultConfig {
    ...
    // This block is different from the one you use to link Gradle
    // to your CMake build script.
    externalNativeBuild {
      cmake {
        ...
        // Use the following syntax when passing arguments to variables:
        // arguments "-DVAR_NAME=VALUE"
        arguments "-DANDROID_ARM_NEON=TRUE"
      }
    }
  }</pre>

## 总结
上面三种姿势肯定推荐第三种， **CMake**  工具确实不错，也是  **AS 2.2**  推出的功能。在实际项目中肯定会有一些这样那样的问题，特别是导入一些像  **FFmpeg**  啊  **x264**  这些外部库的时候，后面会写一些文章把编译  **FFmpeg**  与实际项目中运用的姿势做一个分享，欢迎一起交流。我的  **github**  主页：<a href='https://github.com/mabeijianxi'>https://github.com/mabeijianxi</a>，我的csdn主页：<a href='http://blog.csdn.net/mabeijianxi'>http://blog.csdn.net/mabeijianxi</a>。
