# ffmpeg格式转换裁剪记录
## 前言
因公司项目有需求做格式转换，且需要所集成的SDK较小，故有本记录。
附：需要集成完整ffmpeg库的请移步[ffmpeg-kit](https://github.com/tanersener/ffmpeg-kit)
## 0x00编译
因为需要做裁剪，所以我们要从编译开始
### 环境
我是在macOS下操作，使用脚本[FFmpeg-iOS-build-script](https://github.com/kewlbear/FFmpeg-iOS-build-script)
#### yasm
另外需要安装`yasm`
```
brew install yasm
```
#### gas-preprocessor
##### 获取
1.从github[下载](https://github.com/yuvi/gas-preprocessor )
2.从ffmpeg项目中[获取](https://github.com/FFmpeg/gas-preprocessor)
##### 安装
1.把里边的 `gas-preprocessor.pl` 文件放入 `/usr/local/bin` 里
2.修改文件权限为读和写
3.terminal输入`which gas-preprocessor.pl`可见以下输出即可
```
/usr/local/bin/gas-preprocessor.pl
```
### 开工
1.将项目[FFmpeg-iOS-build-script](https://github.com/kewlbear/FFmpeg-iOS-build-script) clone到本地
```
git clone git@github.com:kewlbear/FFmpeg-iOS-build-script.git
```
2.编辑其中的脚本`build-ffmpeg.sh`
* 指定ffmpeg的版本,这里使用`4.3.1`版本
```
FF_VERSION="4.3.1"
```
* 编译参数,后续会有解释
```
CONFIGURE_FLAGS="--enable-cross-compile --disable-debug --disable-programs \
                 --disable-doc --enable-pic \
                 --disable-programs --disable-ffplay --disable-ffprobe --disable-avdevice --disable-swresample --disable-swscale"
```
3.执行脚本，并等待(此处我只编译了x86_64一种架构，具体怎么用请参照脚本说明)
```
./build-ffmpeg.sh x86_64
```
## 0x01组装
### iOS项目
#### 1.新建iOS项目
#### 2.添加依赖库
找到项目的`Target` > `General` > `Frameworks,Libraries,and Embedded Content`，依次点击`+`号，添加以下依赖
```
AudioToolbox.framework
AVFoundation.framework
VideoToolbox.framework
CoreMedia.framework
libiconv.tbd
libbz2.tbd
libz.tbd
```
#### 3.项目库结构
在项目新建文件夹ffmpeg用于存放库文件,结构如下
```shell
├── ffmpegDemo # 项目根目录
│   ├── ffmpeg 
│   │   ├── fftools # 存放命令行工具
│   │   ├── include # 存放库的头文件
│   │   └── lib # 用于存放库文件
```
将编译脚本中的thin文件中的文件复制到对应的库文件夹,`include`与`lib`一一对应
####4.配置项目搜索路径
因为项目编译的是静态库与头文件，因此，需要设置好头文件搜索路径和静态库文件搜索路径
* 找到项目的`Target` > `Build Settings` > `Header Search Paths`新增一项填入include文件夹的路径，如我的:
```
$(SRCROOT)
```
* 找到项目的`Target` > `Build Settings` > `Library Search Paths`新增一项填入lib文件夹的路径，如我的:
```
$(PROJECT_DIR)/ffmpegDemo/ffmpeg/lib
```
##0x02调用
以下引用自[（五）利用FFmpeg 命令行fftools转码视频](https://www.jianshu.com/p/822c138251a8)

>到这一步其实已经可以使用library库了，如果要对音视频进行操作，需要手动写C++代码去调用 API 使用FFmpeg。你可以导入 #import "avformat.h" 在代码中 写 av_register_all() 然后进行编译，如果没有报错，代表编译成功。av_register_all() 目前版本已经弃用，只是做测试。

>如果想要使用Tool工具来调用 FFmpg 的话，就是直接通过调用传参的方式执行ffmpeg 命令的话，就需要导入对应的文件。

#### fftools
a.可以在`ffmpeg`项目中找到`fftools`文件夹
b.可以在编译脚本的目录下`ffmpeg-4.3.1`文件夹中找到`fftools`文件夹
* 将`fftools`文件夹中的以下文件拷贝到项目库文件夹`fftools`中
```shell
cmdutils.c
cmdutils.h
ffmpeg.c
ffmpeg.h
ffmpeg_filter.c
ffmpeg_hw.c
ffmpeg_opt.c
ffmpeg_videotoolbox.c
```
在编译脚本的目录下的`scratch`文件夹中找到以下文件拷贝到项目库文件夹`fftools`中
```
config.h
```
#### 注释
注释掉一下文件
##### cmdutils.c
```
#include "compat/va_copy.h"
#include "libavresample/avresample.h"
#include "libavutil/libm.h"

PRINT_LIB_INFO(avresample, AVRESAMPLE, flags, level);
```
##### ffmpeg.h

##### ffmpeg_filter.c 
```
#include "libavresample/avresample.h"
```
##### ffmpeg.c
```
#include "libavutil/internal.h"
#include "libavutil/libm.h"

ff_dlog(NULL, "force_key_frame: n:%f n_forced:%f prev_forced_n:%f t:%f prev_forced_t:%f -> res:%f\n",
ost->forced_keyframes_expr_const_values[FKF_N],
ost->forced_keyframes_expr_const_values[FKF_N_FORCED],
ost->forced_keyframes_expr_const_values[FKF_PREV_FORCED_N],
ost->forced_keyframes_expr_const_values[FKF_T],
ost->forced_keyframes_expr_const_values[FKF_PREV_FORCED_T],
res);
```
#### 头文件
基本原则是，`libavcodec`,`libavfilter`,`libavformat`,`libavutil`这四个库缺少什么头文件，就去编译脚本的目录下`ffmpeg-4.3.1`文件夹中找对应库名称的文件夹，将头文件复制到项目库文件夹include/对应的库文件名之下。
例如：
```C
#include "libavcodec/mathops.h" // 报错找不到
```
就去`ffmpeg-4.3.1`中找到`libavcodec`文件夹，并在文件夹中找到`mathops.h`复制到项目库文件夹`libavcodec/`中
## 优化
* 避免main函数重复
```C
// ffmpeg.h 文件下增加函数声明:
int ffmpeg_main(int argc, char **argv);
// ffmpeg.c 文件中:
// main函数修改为ffmpeg_main；主要是为了避免两个main函数存在
int ffmpeg_main(int argc, char **argv){
    //...
}
```
* 计数器置零
```C
//在 ffmpeg.c 中 找到 ffmpeg_cleanup 方法定义，在 term_exit() 前，将各个计数器置零：
nb_filtergraphs=0;
nb_output_files=0;
nb_output_streams=0;
nb_input_files=0;
nb_input_streams=0;

term_exit()
```
* 命令执行结束时崩溃
```
// 在 ffmpeg.c把所有调用 exit_program 函数 ，改为调用 ffmpeg_cleanup 函数就可以了。
```
## 调用
>目前为止，我们做完上面所有步骤后，我们已经可以调用 FFmpeg Tool 进行各种音视频操作了，例如视频合成、视频转Gif、视频帧操作、视频特效、格式转换，视频调速，等各种操作了。Demo里面实现了本文提到的视频转码功能。

>Demo的代码套用网上现有，跟业务相关的需要自己修改。

```objc
//转换视频
- (void)converWithInputPath:(NSString *)inputPath
                 outputPath:(NSString *)outpath
               processBlock:(void (^)(float process))processBlock
            completionBlock:(void (^)(NSError *error))completionBlock {
    self.processBlock = processBlock;
    self.completionBlock = completionBlock;
    self.isBegin = NO;

    // ffmpeg语法，可根据需求自行更改      !#$ 为分割标记符，也可以使用空格代替
    NSString *commandStr = [NSString stringWithFormat:@"ffmpeg!#$-ss!#$00:00:00!#$-i!#$%@!#$-b:v!#$2000K!#$-y!#$%@", inputPath, outpath];

    [[[NSThread alloc] initWithTarget:self selector:@selector(runCmd:) object:commandStr] start];
}


// 执行指令
- (void)runCmd:(NSString *)commandStr{
    // 判断转换状态
    if (self.isRuning) {
        NSLog(@"正在转换,稍后重试");
    }
    self.isRuning = YES;

    // 根据 !#$ 将指令分割为指令数组
    NSArray *argv_array = [commandStr componentsSeparatedByString:(@"!#$")];
    // 将OC对象转换为对应的C对象
    int argc = (int)argv_array.count;
    char** argv = (char**)malloc(sizeof(char*)*argc);
    for(int i=0; i < argc; i++) {
        argv[i] = (char*)malloc(sizeof(char)*1024);
        strcpy(argv[i],[[argv_array objectAtIndex:i] UTF8String]);
    }

    ffmpeg_main(argc,argv);
}
```
### 获取转码进度
* 打开视频源时获取总时长
```C
ffmpeg_opt.c

1、添加头文件 #include "LEYFFmpegConverOC.h"

2、在static int open_input_file(OptionsContext *o, const char *filename)函数的恰当位置添加回调
setDuration(ic->duration);
我放在av_freep(&opts)释放内存之前。
```
* 获取当前转码进度
```C
ffmpeg.c

1、添加头文件 #include "LEYFFmpegConverOC.h"

2、在static void print_report(int is_last_report, int64_t timer_start, int64_t cur_time)函数的恰当位置添加回调
setCurrentTime(buf.str);

我放在 fflush(stderr)语句之前。
```
* 转码结束
```C
ffmpeg.c

1、添加头文件 #include "LEYFFmpegConverOC.h"

2、在ffmpeg_cleanup函数的term_exit()语句之前添加stopRuning();

```