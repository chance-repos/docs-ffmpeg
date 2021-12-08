## 视频转换

比如一个avi文件，想转为mp4，或者一个mp4想转为ts。
`ffmpeg -i input.avi output.mp4`
`ffmpeg -i input.mp4 output.ts`

## 提取音频

```
ffmpeg -i test.mp4 -acodec copy -vn output.aac`
上面的命令，默认mp4的audio codec是aac,如果不是，可以都转为最常见的aac。
`ffmpeg -i test.mp4 -acodec aac -vn output.aac
```

## 设置音频采样率

```
ffmpeg -i a.mp4 -ac 1 -ar 32000 -vcodec copy output.mp4
```

## 提取视频

```
ffmpeg -i input.mp4 -vcodec copy -an output.mp4
```

## 视频解复用

```
ffmpeg –i test.mp4 –vcodec copy –an –f m4v test.264
ffmpeg –i test.avi –vcodec copy –an –f m4v test.264
```
## 视频拼接
### 使用concat协议进行视频文件合并
适用场景：视频容器是MPEG-1, MPEG-2 PS或DV等可以直接进行合并的。换句话说，其实可以直接用cat或者copy之类的命令来对视频直接进行合并。    
1.对于 MPEG 格式的视频，可以直接按文件进行拼接，无需经过编码：
//编码格式、视频尺寸要求一致
```
ffmpeg -i concat:"1.mpg|2.mpg|3.mpg" -c copy output.mp4
```
2.对于非 MPEG 格式容器，但是是 MPEG 编码器(H.264、DivX、XviD、MPEG4、MPEG2、AAC、MP2、MP3 等)，可以包装进 TS 格式的容器再合并。在新浪视频，有很多视频使用 H.264 编码器，可以采用这个方法(视频编码、尺寸要求一致)：
ts全称为MPEG2-TS。ts即"Transport Stream"的缩写。MPEG2-TS格式的特点就是要求从视频流的任一片段开始都是可以独立解码的。
```
ffmpeg -i 1.mp4 -c copy -bsf:v h264_mp4toannexb -f mpegts 1.ts
ffmpeg -i 2.mp4 -c copy -bsf:v h264_mp4toannexb -f mpegts 2.ts
ffmpeg -i 3.mp4 -c copy -bsf:v h264_mp4toannexb -f mpegts 3.ts
ffmpeg -i 'concat:1.ts|2.ts|3.ts' -vcodec copy -acodec copy -absf aac_adtstoasc -movflags +faststart ts.mp4
```
保存 QuickTime/MP4 格式容器的时候，建议加上 -movflags +faststart。这样分享文件给别人的时候可以边下边看。
### 使用FFmpeg concat分离器
适用场景：当容器格式不支持文件层次的合并，而又不想（不需要）进行再编码的操作的时候。这种方式要求源视频编码格式、分辨率、音频采样率等格式相同
FFmpeg concat分离器拼接视频，这种方式不会对视频音频流解码再编码，因此速度很快，成功率很高，质量也是最好的，但是需要FFmpeg 1.1 以上版本。先创建一个文本文件 filelist.txt：
```
file 'input1.mkv'
file 'input2.mkv'
file 'input3.mkv'
```
拼接命令：
```
ffmpeg -f concat -safe 0 -i filelist.txt -c copy output.mp4
```
### 使用concat滤镜(filter)
种方式实际上是把所有的视频音频全部解码，统一为原始的音视频流，然后塞进编码器重新编码。这种方式需要视频之间的分辨率和帧率必须一致，优点是兼容性好，能够应付绝大部分场景，有损压缩
```
ffmpeg -i 11.mp4 -i 22.mp4 -filter_complex "[0:v] [0:a] [1:v] [1:a] concat=n=2:v=1:a=1 [v] [a]" -map "[v]" -map "[a]" output.mp4

[0:0] [0:1] [1:0] [1:1] [2:0] [2:1] 分别表示第一个输入文件的视频、音频、第二个输入文件的视频、音频、第三个输入文件的视频、音频。concat=n=3:v=1:a=1 表示有三个输入文件，输出一条视频流和一条音频流。[v] [a] 就是得到的视频流和音频流的名字，注意在 bash 等 shell 中需要用引号，防止通配符扩展。
```
## 视频封装
```
ffmpeg –i video_file –i audio_file –vcodec copy –acodec copy output_file
```

## 视频剪切

```
ffmpeg –i test.avi –r 1 –f image2 image-%3d.jpeg        //提取图片
ffmpeg -ss 0:1:30 -t 0:0:20 -i input.avi -vcodec copy -acodec copy output.avi    //剪切视频
//-r 提取图像的频率，-ss 开始时间，-t 持续时间
```

## 视频录制

```
ffmpeg –i rtsp://192.168.3.205:5555/test –vcodec copy out.avi
```

## 视频分段切割
```
ffmpeg -i in.mkv -f segment -segment_time 10 -segment_format_options movflags=+faststart out%03d.mp4
```

## 帧率控制

```
ffmpeg –i input.mp4 -r 50 output.mp4
```
注意：帧率的变化会导致码率同步变化，如果需要码率控制，需指定目标码率。

## 码率控制

```
码率控制对于在线视频比较重要。因为在线视频需要考虑其能提供的带宽。
bitrate = file size / duration
比如一个文件20.8M，时长1分钟，那么，码率就是：
biterate = 20.8M bit/60s = 20.8*1024*1024*8 bit/60s= 2831Kbps
一般音频的码率只有固定几种，比如是128Kbps，
那么video的码率video biterate = 2831Kbps -128Kbps = 2703Kbps。

码率控制：码率控制是一种决定为每一个视频帧分配多少比特数的方法，它将决定文件的大小和质量的分配。
码率控制有两种模式：CRF、ABR

### CRF（Constant Rate Factor - 限制码率因子）
适用范围：
优点：该方法在输出文件的大小不太重要的时候，可以使整个文件达到特定的视频质量。该编码模式在单遍编码模式下提供了最大的压缩效率，每一帧可以按照要求的视频质量去获取它需要的比特数。
缺点：不好的一面是，你不能获取一个特定大小的视频文件，或者说将输出位率控制在特定的大小上。

参数解析：
1)量化比例的范围为0～51，其中0为无损模式，23为缺省值，51可能是最差的。该数字越小，图像质量越好。从主观上讲，18~28是一个合理的范围。18往往被认为从视觉上看是无损的，它的输出视频质量和输入视频一模一样或者说相差无几。但从技术的角度来讲，它依然是有损压缩。
2)若Crf值加6，输出码率大概减少一半；若Crf值减6，输出码率翻倍。通常是在保证可接受视频质量的前提下选择一个最大的Crf值，如果输出视频质量很好，那就尝试一个更大的值，如果看起来很糟，那就尝试一个小一点值。

使用方法 - 命令行：
ffmpeg -i <input> -c:v libx264 -crf 23 <output>
ffmpeg -i <input> -c:v libx265 -crf 28 <output>
ffmpeg -i <input> -c:v libvpx-vp9 -crf 30 -b:v 0 <output>

### ABR（Average Bitrate - 平均码率）
解析：

它提供了某种“运行均值”的目标，终极目标是最终文件大小匹配这个“全局平均”数字（因此基本上来说，如果编码器遇到大量码率开销非常小的黑帧，它将以低于要求的比特率编码，但是在接下来几秒内的非黑帧它将以高质量方式编码方式使码率回归均值）使用两遍编码模式时这个方法变得更加有效，你可以和“max bit rate ”配合使用来防止码率的波动。

使用方法 - 命令行

ffmpeg -i <input> -c:v libx264 -b:v 1M <output>
ffmpeg -i <input> -c:v libx265 -b:v 1M <output>
ffmpeg -i <input> -c:v libvpx-vp9 -b:v 1M <output>

-b:v主要是控制平均码率。
比如一个视频源的码率太高了，有10Mbps，文件太大，想把文件弄小一点，但是又不破坏分辨率。
`ffmpeg -i input.mp4 -b:v 2000k output.mp4`
上面把码率从原码率转成2Mbps码率，这样其实也间接让文件变小了。目测接近一半。
不过，ffmpeg官方wiki比较建议，设置b:v时，同时加上 -bufsize
-bufsize 用于设置码率控制缓冲器的大小，设置的好处是，让整体的码率更趋近于希望的值，减少波动。（简单来说，比如1 2的平均值是1.5， 1.49 1.51 也是1.5, 当然是第二种比较好）
`ffmpeg -i input.mp4 -b:v 2000k -bufsize 2000k output.mp4`

-minrate -maxrate就简单了，在线视频有时候，希望码率波动，不要超过一个阈值，可以设置maxrate。
`ffmpeg -i input.mp4 -b:v 2000k -bufsize 2000k -maxrate 2500k output.mp4`

使用方法(源码)
如果设置了AVCodecContext中bit_rate的大小，则采用abr的控制方式
如果没有设置AVCodecContext中的bit_rate，则默认按照crf方式编码，crf默认大小为23

如果需要设置恒定码率，可以通过补充ABR参数“模拟”一个恒定比特率设置：

使用方法 - 命令行
// H264
ffmpeg -i <input> -c:v libx264 -x264-params "nal-hrd=cbr:force-cfr=1" -b:v 1M -minrate 1M -maxrate 1M -bufsize 2M <output>
// VP9
ffmpeg -i <input> -c:v libvpx-vp9 -b:v 1M -maxrate 1M -minrate 1M <output>

几个关注点：
1、视频文件码率 = 音频码率* 音频压缩比率 + 视频图像码率* 视频压缩比率，这个结果是个大致值，还受其他因素影响
2、帧率 * 像素宽 * 像素长 * 位深度 = 视频图像码率。如一个视频属性为：60fps（帧率），480*720p，32位位深，则视频图像码率 = 60 * 480 * 720 * 32
3、ffmpeg的码率转换关系：1M=1000kb=1000*1000bit
```



## 视频编码格式转换

```
视频的编码MPEG4转H264编码
`ffmpeg -i input.mp4 -vcodec h264 output.mp4`
相反也一样
`ffmpeg -i input.mp4 -vcodec mpeg4 output.mp4`

当然了，如果ffmpeg当时编译时，添加了外部的x265或者X264，那也可以用外部的编码器来编码。（不知道什么是X265，可以 Google一下，简单的说，就是她不包含在ffmpeg的源码里，是独立的一个开源代码，用于编码HEVC，ffmpeg编码时可以调用它。当然 了，ffmpeg自己也有编码器）
`ffmpeg -i input.mp4 -c:v libx265 output.mp4`
`ffmpeg -i input.mp4 -c:v libx264 output.mp4`


ffmpeg –i test.mp4 –vcodec h264 –s 352*278 –an –f m4v test.264              //转码为码流原始文件
ffmpeg –i test.mp4 –vcodec h264 –bf 0 –g 25 –s 352*278 –an –f m4v test.264  //转码为码流原始文件
ffmpeg –i test.avi -vcodec mpeg4 –vtag xvid –qsame test_xvid.avi            //转码为封装文件
//-bf B帧数目控制，-g 关键帧间隔控制，-s 分辨率控制
```



## 只提取视频ES数据

```
ffmpeg –i input.mp4 –vcodec copy –an –f m4v output.h264
```

## 过滤器的使用

### 将输入的1920x1080缩小到960x540输出:

`ffmpeg -i input.mp4 -vf scale=960:540 output.mp4`
//ps: 如果540不写，写成-1，即scale=960:-1, 那也是可以的，ffmpeg会通知缩放滤镜在输出时保持原始的宽高比。

### FFmpeg给视频添加黑边
使用FFmpeg给视频增加黑边需要用到pad这个滤镜，具体用法如下：
```
-vf pad=1280:720:0:93:black
```
按照从左到右的顺序依次为:
​“宽”、“高”、“X坐标”和“Y坐标”，宽和高指的是输入视频尺寸（包含加黑边的尺寸），XY指的是视频所在位置。
比如一个输入视频尺寸是1280x534的源，想要加上黑边变成1280x720，那么用上边的语法可以实现，93是这样得来的，（720-534）/2。
如果视频原始1920x800的话，完整的语法应该是：
```
    -vf 'scale=1280:534,pad=1280:720:0:93:black'
```
先将视频缩小到1280x534，然后在加入黑边变成1280x720，将1280x534的视频放置在x=0，y=93的地方，
​FFmpeg会自动在上下增加93像素的黑边。
注：black可以不写，默认是黑色

### 为视频添加logo

比如，我有这么一个图片
![iqiyi logo](http://img.blog.csdn.net/20160512155254687)
想要贴到一个视频上，那可以用如下命令：
./ffmpeg -i input.mp4 -i iQIYI_logo.png -filter_complex overlay output.mp4
结果如下所示：
![add logo left](http://img.blog.csdn.net/20160512155411797)
要贴到其他地方？看下面：
右上角：
./ffmpeg -i input.mp4 -i logo.png -filter_complex overlay=W-w output.mp4
左下角：
./ffmpeg -i input.mp4 -i logo.png -filter_complex overlay=0:H-h output.mp4
右下角：
./ffmpeg -i input.mp4 -i logo.png -filter_complex overlay=W-w:H-h output.mp4

### 去掉视频的logo

语法：-vf delogo=x:y:w:h[:t[:show]]
x:y 离左上角的坐标
w:h logo的宽和高
t: 矩形边缘的厚度默认值4
show：若设置为1有一个绿色的矩形，默认值0。

`ffmpeg -i input.mp4 -vf delogo=0:0:220:90:100:1 output.mp4`
结果如下所示：
![de logo](http://img.blog.csdn.net/20160512155451204)

## 截取视频图像

`ffmpeg -i input.mp4 -r 1 -q:v 2 -f image2 pic-%03d.jpeg`
-r 表示每一秒几帧
-q:v表示存储jpeg的图像质量，一般2是高质量。
如此，ffmpeg会把input.mp4，每隔一秒，存一张图片下来。假设有60s，那会有60张。



可以设置开始的时间，和你想要截取的时间。
`ffmpeg -i input.mp4 -ss 00:00:20 -t 10 -r 1 -q:v 2 -f image2 pic-%03d.jpeg`
-ss 表示开始时间
-t 表示共要多少时间。
如此，ffmpeg会从input.mp4的第20s时间开始，往下10s，即20~30s这10秒钟之间，每隔1s就抓一帧，总共会抓10帧。

## 序列帧与视频的相互转换

把darkdoor.[001-100].jpg序列帧和001.mp3音频文件利用mpeg4编码方式合成视频文件darkdoor.avi：
**$ ffmpeg -i 001.mp3 -i darkdoor.%3d.jpg -s 1024x768 -author fy -vcodec mpeg4 darkdoor.avi**

还可以把视频文件导出成jpg序列帧：
$ ***\*ffmpeg -i bc-cinematic-en.avi example.%d.jpg\****

------

# 其他用法

## 输出YUV420原始数据

对于一下做底层编解码的人来说，有时候常要提取视频的YUV原始数据，如下：
`ffmpeg -i input.mp4 output.yuv`

**那如果我只想要抽取某一帧YUV呢？**
你先用上面的方法，先抽出jpeg图片，然后把jpeg转为YUV。
比如：
你先抽取10帧图片。

```
ffmpeg -i input.mp4 -ss 00:00:20 -t 10 -r 1 -q:v 2 -f image2 pic-%03d.jpeg
```

然后，你就随便挑一张，转为YUV:
`ffmpeg -i pic-001.jpeg -s 1440x1440 -pix_fmt yuv420p xxx3.yuv`
如果-s参数不写，则输出大小与输入一样。

当然了，YUV还有yuv422p啥的，你在-pix_fmt 换成yuv422p就行啦！

## H264编码profile & level控制

### 背景知识

先科普一下profile&level。（这里讨论最常用的H264）
H.264有四种画质级别,分别是baseline, extended, main, high：
　　1、Baseline Profile：基本画质。支持I/P 帧，只支持无交错（Progressive）和CAVLC；
　　2、Extended profile：进阶画质。支持I/P/B/SP/SI 帧，只支持无交错（Progressive）和CAVLC；(用的少)
　　3、Main profile：主流画质。提供I/P/B 帧，支持无交错（Progressive）和交错（Interlaced），
　　　 也支持CAVLC 和CABAC 的支持；
　　4、High profile：高级画质。在main Profile 的基础上增加了8x8内部预测、自定义量化、 无损视频编码和更多的YUV 格式；
H.264 Baseline profile、Extended profile和Main profile都是针对8位样本数据、4:2:0格式(YUV)的视频序列。在相同配置情况下，High profile（HP）可以比Main profile（MP）降低10%的码率。
根据应用领域的不同，Baseline profile多应用于实时通信领域，Main profile多应用于流媒体领域，High profile则多应用于广电和存储领域。

下图清楚的给出不同的profile&level的性能区别。
**profile**
![这里写图片描述](http://img.blog.csdn.net/20160516164141047)

**level**
![这里写图片描述](http://img.blog.csdn.net/20160516164454535)

### ffmpeg如何控制profile&level
`ffmpeg -i input.mp4 -profile:v baseline -level 3.0 output.mp4`

```
ffmpeg -i input.mp4 -profile:v main -level 4.2 output.mp4
ffmpeg -i input.mp4 -profile:v high -level 5.1 output.mp4
```

如果ffmpeg编译时加了external的libx264，那就这么写：
`ffmpeg -i input.mp4 -c:v libx264 -x264-params "profile=high:level=3.0" output.mp4`

从压缩比例来说，baseline< main < high，对于带宽比较局限的在线视频，可能会选择high，但有些时候，做个小视频，希望所有的设备基本都能解码（有些低端设备或早期的设备只能解码 baseline），那就牺牲文件大小吧，用baseline。自己取舍吧！

苹果的设备对不同profile的支持。
![这里写图片描述](http://img.blog.csdn.net/20160516171746876)

### 编码效率和视频质量的取舍(preset, crf)

除了上面提到的，强行配置biterate，或者强行配置profile/level，还有2个参数可以控制编码效率。
一个是preset，一个是crf。
preset也挺粗暴，基本原则就是，如果你觉得编码太快或太慢了，想改改，可以用profile。
preset有如下参数可用：

> ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow and placebo.
> 编码加快，意味着信息丢失越严重，输出图像质量越差。

CRF(Constant Rate Factor): 范围 0-51: 0是编码毫无丢失信息, 23 is 默认, 51 是最差的情况。相对合理的区间是18-28.
值越大，压缩效率越高，但也意味着信息丢失越严重，输出图像质量越差。

举个例子吧。
`ffmpeg -i input -c:v libx264 -profile:v main -preset:v fast -level 3.1 -x264opts crf=18`
(参考自：https://trac.ffmpeg.org/wiki/Encode/H.264)

#### H265 (HEVC)编码tile&level控制

##### 背景知识

和H264的profile&level一样，为了应对不同应用的需求，HEVC制定了“层级”(tier) 和“等级”(level)。
tier只有main和high。
level有13级，如下所示：
![这里写图片描述](http://img.blog.csdn.net/20160516181155149)

```
ffmpeg -i input.mp4 -c:v libx265 -x265-params "profile=high:level=3.0" output.mp4
```

