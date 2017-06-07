## 本人环境与工具:<br>
> * 系统：MacOs-10.12.4<br>
> * ndk：r14<br>
> * FFmpeg版本: 3.2.5<br>
> * Android Studio: 2.3.2

## 一、说明：
本文是经过实战总结出的经验，本文将用两种方式编译可以在Android下执行命令的FFmpeg,一种是传统的**ndk-build**工具，一种是**cmake**工具，经过我的项目实战，非常推荐cmake，因为**AS 2.2**以后对它支持的非常好，你可以非常方便的像debug Java代码一样去debug Native代码。还有在看本文以前建议先看[Android下玩JNI的新老三种姿势](http://blog.csdn.net/mabeijianxi/article/details/68525164)与<a href='http://blog.csdn.net/mabeijianxi/article/details/72888067'> 编译Android下可用的FFmpeg(包含libx264与libfdk-aac)，后文有些东西在这两篇文章里面记录过了就不再重复。
 </a>。<br>
 
## 二、传统ndk-build命令编译
所谓传统，必然就稍微没那么智能了，我们一步一步的搞。<br>

1. 打开你养家糊口的Android Studio,娴熟的新建一个项目;<br>
2. 编写一个 **native** 函数，如果只是测试我们在MainActivity里面搞就行了：<br><br>
<pre><code>
public native int ffmpegRun(String[] cmd);
</code></pre><br<br>

3. 新建jni目录，在目录下新建文件: **jx\_ffmpeg_cmd\_run.c**;<br>
4. 编码对应的 JNI 接口：
<pre><code>
\#include <jni.h><br>
JNIEXPORT jint JNICALL
Java\_com\_mabeijianxi\_jianxiffmpegcmd\_MainActivity\_ffmpegRun(JNIEnv *env, jobject instance, jobjectArray cmd) {

    // TODO
}</code></pre><br<br>
5. 找到我们FFmpeg编译后的根目录，然后 copy：<br>
**cmdutils.c  cmdutils.h cmdutils\_common_opts.h config.h ffmpeg.c ffmpeg.h ffmpeg\_filter.c ffmpeg\_opt.c**（注意需要编译后才会有config.h）到 jni 目录下,再进入到我们的编译后的产物目录，把**include**文件夹与所有的 **.so** 动态库也 copy 到jni目录下。完成后你jni目录结构应该如下图:<br><br>
![pic_3.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_3.png)<br><br>
6. 文件修改<br>
修改ffmpeg.c与ffmpeg.h<br>
找到ffmpeg.c，把int main(int argc, char argv) 改名为 int jxRun(int argc, char argv)
找到ffmpeg.h, 在文件末尾添加函数申明: int jxRun(int argc, char \**argv);
<br><br>![pic_4.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_4.png)<br><br>
1)修改cmdutils.c 和 cmdutils.h<br>
找到cmdutils.c中的exit\_program函数<br>
修改前:
<br><br>![pic_5.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_5.png)<br><br>
修改后:
<br><br>![pic_6.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_6.png)<br><br>
2)找到cmdutils.h中exit_program的申明，也把返回类型修改为int。<br>
修改前:
<br><br>![pic_7.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_7.png)<br><br>
修改后:
<br><br>![pic_8.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_8.png)<br><br>
很多教程都只修改到这里，基本没什么问题，但是你实际运行的时候会发现如果连续多次执行命令会有问题的，通过源码我们可以知道，FFmpeg每次执行完命令后会调用 **ffmpeg\_cleanup** 函数清理内存，并且会调用exit(0)结束当前进程，但是经过我们的修改，exit()的代码已经被删掉，我们在Android中自然不能结束当前进程了，所以有些变量的值还在内存中，这样就会导致下次执行的时候可能会出错。我也尝试过fork一个进程给ffmpeg执行，完事后通过 **信号**来进程间通信，这样管用但是很麻烦，我们其实只需要简单的重设一些变量即可。<br>
打开ffmpeg.c找到刚修改的**jxRun**函数，然后在 **return** 前加上如下代码即可:<br>
<pre><code>
nb\_filtergraphs = 0;
progress\_avio = NULL;<br>
input\_streams = NULL;
nb\_input\_streams = 0;
input\_files = NULL;
nb\_input\_files = 0;<br>
output\_streams = NULL;
nb\_output\_streams = 0;
output\_files = NULL;
nb\_output\_files = 0;
</code></pre><br>

7. 编写调用函数<br>
我们上面只在**jx\_ffmpeg_cmd\_run.c**新建了一个JNI接口函数，还没有实现逻辑，我们实现后的代码如下：<br>
<pre><code>
/\**
 \* Created by jianxi on 2017/6/4.
 \* https://github.com/mabeijianxi
 \* mabeijianxi@gmail.com
 \*/
\#include "ffmpeg.h"
\#include \<jni.h\><br>
JNIEXPORT jint JNICALL
Java\_com\_mabeijianxi\_jianxiffmpegcmd\_MainActivity\_ffmpegRun(JNIEnv \*env, jobject type,jobjectArray commands){
    int argc = (\*env)->GetArrayLength(env,commands);
    char \*argv[argc];
    int i;
    for (i = 0; i < argc; i++) {
        jstring js = (jstring) (\*env)->GetObjectArrayElement(env,commands, i);
        argv[i] = (char \*) (\*env)->GetStringUTFChars(env,js, 0);
    }
    return jxRun(argc,argv);
}
</code></pre>
8. 编写**Application.mk**与**Android.mk**<br>
1)在jni目录下新建**Application.mk**与**Android.mk**<br>
2)编写**Application.mk**,内容如下:<br>
<pre><code>
APP_ABI := armeabi-v7a
APP_PLATFORM := android-14
</code></pre>
3)编码**Android.mk**，其内容如下（不明含义的可看[Android下玩JNI的新老三种姿势](http://blog.csdn.net/mabeijianxi/article/details/68525164)）<br>
<pre><code>
LOCAL\_PATH := $(call my-dir)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libavcodec
LOCAL\_SRC\_FILES := libavcodec.so
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libavfilter
LOCAL\_SRC\_FILES := libavfilter.so(https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libavformat
LOCAL\_SRC\_FILES := libavformat.so
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libavutil
LOCAL\_SRC\_FILES := libavutil.so
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libswresample
LOCAL\_SRC\_FILES := libswresample.so
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := libswscale
LOCAL\_SRC\_FILES := libswscale.so
include $(PREBUILT\_SHARED\_LIBRARY)<br>
include $(CLEAR\_VARS)
LOCAL\_MODULE := jxffmpegrun
LOCAL\_SRC\_FILES := cmdutils.c ffmpeg.c ffmpeg\_filter.c ffmpeg\_opt.c jx\_ffmpeg\_cmd\_run.c<br>
\# 这里的地址改成自己的 FFmpeg 源码目录
LOCAL\_C\_INCLUDES :=/Users/jianxi/Downloads/code/ffmpeg-3.2.5
LOCAL\_LDLIBS := -llog -lz -ldl
LOCAL\_SHARED\_LIBRARIES :=libavcodec libavfilter libavformat libavutil libswresample libswscale
include $(BUILD\_SHARED\_LIBRARY)
</code></pre><br>
9. 开始编译:<br>
我们打开终端，然后cd 到jni目录下，然后执行**ndk-build**，如果你没配置其环境变量，你选择写入**ndk-build**的全路径。<br>
如果顺利你可以在命令结束后看到如下输出:<br>
<br>
![pic_9.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_9.png)
<br><br>
并且你的lib下会有如下产物:<br>
<br>
![pic_10.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_10.png)
<br><br>
这里就先不上测试的效果图了，等后面一起搞<br>

## 三、cMake编译
有好多步骤都是很上面一样的，一会儿我就快乐的copy下来哈<br>

1. 打开你养家糊口的Android Studio,版本最好大于2.2，很关键。娴熟的新建一个项目，但是与以往不同，你最好勾选上 **C++** 支持与 **C++ standard**选项时选择 **C++ 11**,如下图：<br><br>
![pic_1.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_1.png)<br><br>
![pic_2.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_2.png)<br><br>
2. 新建完成后你会发现帮你生成好了接口，这时我们只需修改**native**函数即可：<br><br>
修改前：<br>
<pre><code>
public native String stringFromJNI();
</code></pre><br<br>
修改后<br>
<pre><code>
public native int ffmpegRun(String[] cmd);
</code></pre><br<br>

3. 修改native文件与函数:<br>
 你会发现这时多了一个cpp的文件夹，里面多了一个**native-lib.cpp**的文件,我们修改其名为**jx\_ffmpeg_cmd\_run.c**，然后修改里面的函数，修改后的函数应该如下:<br>
<pre><code>
\#include <jni.h><br>
JNIEXPORT jint JNICALL
Java\_com\_mabeijianxi\_jianxiffmpegcmd\_MainActivity\_ffmpegRun(JNIEnv *env, jobject instance, jobjectArray cmd) {

    // TODO
}</code></pre><br<br>
4. 找到我们FFmpeg编译后的根目录，然后 copy：<br>
**cmdutils.c  cmdutils.h cmdutils\_common_opts.h config.h ffmpeg.c ffmpeg.h ffmpeg\_filter.c ffmpeg\_opt.c**（注意需要编译后才会有config.h）到 cpp 目录下,再进入到我们的编译后的产物目录，把**include**文件夹与所有的 **.so** 动态库也 copy 到cpp目录下。完成后你cpp目录结构应该如下图:<br><br>
![pic_11.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_11.png)<br><br>
5. 文件修改<br>
修改ffmpeg.c与ffmpeg.h<br>
找到ffmpeg.c，把int main(int argc, char argv) 改名为 int jxRun(int argc, char argv)
找到ffmpeg.h, 在文件末尾添加函数申明: int jxRun(int argc, char \**argv);
<br><br>![pic_4.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_4.png)<br><br>
1)修改cmdutils.c 和 cmdutils.h<br>
找到cmdutils.c中的exit\_program函数<br>
修改前:
<br><br>![pic_5.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_5.png)<br><br>
修改后:
<br><br>![pic_6.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_6.png)<br><br>
2)找到cmdutils.h中exit_program的申明，也把返回类型修改为int。<br>
修改前:
<br><br>![pic_7.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_7.png)<br><br>
修改后:
<br><br>![pic_8.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_8.png)<br><br>
很多教程都只修改到这里，基本没什么问题，但是你实际运行的时候会发现如果连续多次执行命令会有问题的，通过源码我们可以知道，FFmpeg每次执行完命令后会调用 **ffmpeg\_cleanup** 函数清理内存，并且会调用exit(0)结束当前进程，但是经过我们的修改，exit()的代码已经被删掉，我们在Android中自然不能结束当前进程了，所以有些变量的值还在内存中，这样就会导致下次执行的时候可能会出错。我也尝试过fork一个进程给ffmpeg执行，完事后通过 **信号**来进程间通信，这样管用但是很麻烦，我们其实只需要简单的重设一些变量即可。<br>
打开ffmpeg.c找到刚修改的**jxRun**函数，然后在 **return** 前加上如下代码即可:<br>
<pre><code>
nb\_filtergraphs = 0;
progress\_avio = NULL;<br>
input\_streams = NULL;
nb\_input\_streams = 0;
input\_files = NULL;
nb\_input\_files = 0;<br>
output\_streams = NULL;
nb\_output\_streams = 0;
output\_files = NULL;
nb\_output\_files = 0;
</code></pre><br>

6. 编写调用函数<br>
我们上面只在**jx\_ffmpeg_cmd\_run.c**新建了一个JNI接口函数，还没有实现逻辑，我们实现后的代码如下：<br>
<pre><code>
/\**
 \* Created by jianxi on 2017/6/4.
 \* https://github.com/mabeijianxi
 \* mabeijianxi@gmail.com
 \*/
\#include "ffmpeg.h"
\#include \<jni.h\><br>
JNIEXPORT jint JNICALL
Java\_com\_mabeijianxi\_jianxiffmpegcmd\_MainActivity\_ffmpegRun(JNIEnv \*env, jobject type,jobjectArray commands){
    int argc = (\*env)->GetArrayLength(env,commands);
    char \*argv[argc];
    int i;
    for (i = 0; i < argc; i++) {
        jstring js = (jstring) (\*env)->GetObjectArrayElement(env,commands, i);
        argv[i] = (char \*) (\*env)->GetStringUTFChars(env,js, 0);
    }
    return jxRun(argc,argv);
}
</code></pre>
7. 编写cmake编译脚本:<br>
	在没有编写脚本的时候你的代码应该是一片红，没关系，马上我们就干掉它。<br>
	打开你当前Module下的**CMakeLists.txt**然后填写如下脚本(内容解释可看[Android下玩JNI的新老三种姿势](http://blog.csdn.net/mabeijianxi/article/details/68525164)):<br>
	<pre><code>
	\# For more information about using CMake with Android Studio, read the
	\# documentation: https://d.android.com/studio/projects/add-native-code.html<br>
	\# Sets the minimum version of CMake required to build the native library.<br>
cmake_minimum_required(VERSION 3.4.1)<br><br>
\# Creates and names a library, sets it as either STATIC
\# or SHARED, and provides the relative paths to its source code.
\# You can define multiple libraries, and CMake builds them for you.
\# Gradle automatically packages shared libraries with your APK.<br>
add_library( # Sets the name of the library.<br>
             jxffmpegrun
             \# Sets the library as a shared library.
             SHARED
             \# Provides a relative path to your source file(s).
              src/main/cpp/cmdutils.c
              src/main/cpp/ffmpeg.c
              src/main/cpp/ffmpeg_filter.c
              src/main/cpp/ffmpeg_opt.c
              src/main/cpp/jx_ffmpeg_cmd_run.c
             )
add_library(
            avcodec
            SHARED
            IMPORTED
            )
set_target_properties(
    		avcodec
    		PROPERTIES IMPORTED_LOCATION
    		/Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libavcodec.so
    )
add_library(
            avfilter
            SHARED
            IMPORTED
             )
set_target_properties(
        	avfilter
        	PROPERTIES IMPORTED_LOCATION
       	/Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libavfilter.so
        )
add_library(
            avformat
            SHARED
            IMPORTED
            )
set_target_properties(
            avformat
            PROPERTIES IMPORTED_LOCATION
            /Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libavformat.so
            )
add_library(
            avutil
            SHARED
            IMPORTED
            )
set_target_properties(
            avutil
            PROPERTIES IMPORTED_LOCATION
            /Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libavutil.so
            )
add_library(
            swresample
            SHARED
            IMPORTED
            )
set_target_properties(
            swresample
            PROPERTIES IMPORTED_LOCATION
            /Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libswresample.so
             )
add_library(
            swscale
            SHARED
            IMPORTED
            )
set_target_properties(
            swscale
            PROPERTIES IMPORTED_LOCATION
            /Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libswscale.so
             )
add_library(
            jxffmpegcmd
            SHARED
            IMPORTED
            )
set_target_properties(
            jxffmpegcmd
            PROPERTIES IMPORTED_LOCATION
            /Users/jianxi/android/projects/JianXIFFmpegCMD/app/src/main/cpp/libjxffmpegrun.so
             )<br>
include_directories(
    /Users/jianxi/Downloads/code/ffmpeg-3.2.5/
)<br>
\# Searches for a specified prebuilt library and stores the path as a
\# variable. Because CMake includes system libraries in the search path by
\# default, you only need to specify the name of the public NDK library
\# you want to add. CMake verifies that the library exists before
\# completing its build.<br>
find_library( \# Sets the name of the path variable.
              log-lib<br>
              \# Specifies the name of the NDK library that
              \# you want CMake to locate.
              log )
\# Specifies libraries CMake should link to your target library. You
\# can link multiple libraries, such as libraries you define in this
\# build script, prebuilt third-party libraries, or system libraries.<br>
target_link_libraries( # Specifies the target library.
                       jxffmpegrun
                       avcodec
                       avfilter
                       avformat
                       avutil
                       swresample
                       swscale
                       \# Links the target library to the log library
                       \# included in the NDK.
                       ${log-lib} )
	</code></pre><br>
	当然你还需要修改脚本里面的一些文件路径。<br>
8. 修改当前Module下的**build.gradle**脚本:<br>
我们编译的FFmpeg是ARM-v7的，所以我们还需要添加过滤:<br>
修改前:
<br><br>![pic_12.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_12.png)<br><br>
修改后:
<br><br>![pic_13.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_13.png)<br><br>

做到这步我们就可以直接点击同步按钮了:
<br><br>![pic_14.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_14.png)<br><br>
完成后就不会再爆红了，这时候是只能直接运行安装的，但是还没有输入命令，所以没什么意义，接下来将开始使用我们做好的工具。<br>

## 四、测试与使用
我们要用动态库肯定得放入默认目录，或者在**gradle.build**中修改其路径，这里我就直接放入jniLibs中的armeabi-v7a里面了，如图:
<br><br>![pic_15.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_15.png)<br><br>
然后我就是在java里面调用了，搞了一个进度条，一个按钮。没有什么技术含量，直接贴代码了哈：<br>
<pre><code>
public class MainActivity extends AppCompatActivity {<br>
    static {
        System.loadLibrary("jxffmpegrun");
        System.loadLibrary("avcodec");
        System.loadLibrary("avfilter");
        System.loadLibrary("avformat");
        System.loadLibrary("avutil");
        System.loadLibrary("swresample");
        System.loadLibrary("swscale");
    }
    private ProgressBar pb;<br>
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        pb = (ProgressBar) findViewById(R.id.pb);
    }<br>
    public void onClick(View v){
        pb.setVisibility(View.VISIBLE);
        new Thread(new Runnable() {
            @Override
            public void run() {
                String basePath = Environment.getExternalStorageDirectory().getPath();
                String cmd_transcoding = String.format("ffmpeg -i %s -c:v libx264 %s  -c:a libfdk_aac %s",
                        basePath+"/"+"girl.mp4",
                        "-crf 40",
                        basePath+"/"+"my\_girl.mp4");
                int i = jxFFmpegCMDRun(cmd_transcoding);
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        pb.setVisibility(View.GONE);
                        Toast.makeText(MainActivity.this,"ok了",Toast.LENGTH\_SHORT).show();
                    }
                });
            }
        }).start();
    }<br>
    public  int jxFFmpegCMDRun(String cmd){
        String regulation="[ \\\\t]+";
        final String[] split = cmd.split(regulation);
        return ffmpegRun(split);
    }<br>
    public native int ffmpegRun(String[] cmd);
}
</code></pre><br>
再上面代码中我们指定编码器压缩了一个**mp4**的文件，压缩前叫**girl.mp4**,压缩后叫**my\_girl.mp4**,如图所示，我们发现其缩小了近**80%**。
<br><br>![pic_16.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_16.png)<br><br>

## 五、总结
这里输入路径不能每次是同一个，不然会报错，也就是说，运行了一次上面的命令后，输出的名字不能再叫**my\_girl.mp4**了，你可以叫**your_girl.mp4**。然后还是那句话推荐用cMake,你看我们通过cMake编译可以直接debug native<br>
<br><br>![pic_17.png](https://raw.githubusercontent.com/mabeijianxi/pic-trusteeship/master/pic/ffmpeg_cmd/pic_17.png)<br><br>
是不是很hi，然后我们在开发中可能会发现有些命令执行不了，这时候你首先需要检查命令，然后确保没问题后，你需要确定你在编译**FFmpeg**的时候是否开启了此功能，很关键，确定办法很简单，一是检查FFmpeg的编译脚本，二是通过**FFmpeg**的函数获取编译信息，我工程里面已经内含这个jni接口，有兴趣的基友可以下载来try一try。如果有兴趣进一步探索FFmpeg在Android上的实战运用，可以看我的开源项目和其讲解博客，分别是：[https://github.com/mabeijianxi/small-video-record](https://github.com/mabeijianxi/small-video-record)、[利用FFmpeg玩转Android视频录制与压缩（一）](http://blog.csdn.net/mabeijianxi/article/details/63335722)、[利用FFmpeg玩转Android视频录制与压缩（二）]()。
