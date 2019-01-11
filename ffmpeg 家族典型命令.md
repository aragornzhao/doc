# ffmpeg 家族典型命令

@(开发)[音视频]

### 锐化（模糊）
```bash
ffmpeg.exe -i original.mp4 -vf unsharp=5:5:-2 blur.mp4
```
unsharp 5:5:-2 , 3个参数，前两个表示处理block，最后一个代表程度，负数表示模糊
### 计算ssim与psnr
```bash
ffmpeg -i blur.mp4 -i original.mp4 -lavfi "ssim;[0:v][1:v]psnr" -f null -
```
###截取视频
```bash
ffmpeg.exe -ss 00:00:03 -t 5 -i original.mp4 -vcodec copy -an cut.mp4
```
-ss 开始时间， -t 结束时间，  -an 不使能音频

###获取分辨率
```bash
ffprobe.exe -v quiet -print_format json -show_format -show_streams original.mp4

```
### 获取视频时长
```bash
ffprobe -show_format video.mp4 
```
duration 为视频时长

### 获取视频帧时间戳
```bash
ffprobe -show_format video.mp4 
```
pkt_pts_time

