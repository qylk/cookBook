# 视频压缩
```bash
ffmpeg 
-i input.mp4 ##输入
-c:v libx264 ##视频编码
-preset slow  ##预置速度, 越慢画质越好, 可选ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow, placebo
-c:a copy ##音频编码
-b:v 3000k ##视频输出码率
-r 30 ##输出帧率
-crf 18 ##动态码率，18~28是一个合理的范围, 18被认为是视觉无损的.
-fs 10MB  ##输出文件大小指定
-s 500×500 #输出视频尺寸
-threads 1 #处理线程数，影响CPU消耗
output.mp4 ##输出
```

ffmpeg下载：<https://evermeet.cx/ffmpeg/>