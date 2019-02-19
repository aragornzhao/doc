# python进行语音信号分析

@(生活)[频谱分析]

## 相关原理
### 频谱分析原理
采样定理：采样频率要大于信号频率的两倍。
N个采样点经过FFT变换后得到N个点的以复数形式记录的FFT结果。
假设采样频率为Fs，采样点数为N。那么FFT运算的结果就是N个复数（或N个点），每一个复数就对应着一个频率值以及该频率信号的幅值和相位。第一个点对应的频率为0Hz（即直流分量），最后一个点N的下一个点对应采样频率Fs。其中任意一个采样点n所代表的信号频率：
Fn=(n-1)*Fs/N。

这表明，频谱分析得到的信号频率最大为(N-1)*Fs/N,对频率的分辨能力是Fs/N。采样频率和采样时间制约着通过FFT运算能分析得到的信号频率上限，同时也限定了分析得到的信号频率的分辨率

### 信号加窗
分析时是需要截断的，截断可能不是周期的整数倍，这就会造成频率泄露。一个优化措施是窗函数
信号加窗代码
```python
def enframe(signal, nw, inc, winfunc):
    '''将音频信号转化为帧。
    参数含义：
    signal:原始音频型号
    nw:每一帧的长度(这里指采样点的长度，即采样频率乘以时间间隔)
    inc:相邻帧的间隔（同上定义）
    '''
    signal_length=len(signal) #信号总长度
    if signal_length<=nw: #若信号长度小于一个帧的长度，则帧数定义为1
        nf=1
    else: #否则，计算帧的总长度
        nf=int(np.ceil((1.0*signal_length-nw+inc)/inc))
    pad_length=int((nf-1)*inc+nw) #所有帧加起来总的铺平后的长度
    zeros=np.zeros((pad_length-signal_length,)) #不够的长度使用0填补，类似于FFT中的扩充数组操作
    pad_signal=np.concatenate((signal,zeros)) #填补后的信号记为pad_signal
    indices=np.tile(np.arange(0,nw),(nf,1))+np.tile(np.arange(0,nf*inc,inc),(nw,1)).T  #相当于对所有帧的时间点进行抽取，得到nf*nw长度的矩阵
    indices=np.array(indices,dtype=np.int32) #将indices转化为矩阵
    frames=pad_signal[indices] #得到帧信号
    win=np.tile(winfunc,(nf,1))  #window窗函数，这里默认取1
    return frames*win   #返回帧信号矩阵
```
使用，以hamming窗为例
```python
winfunc = signal.hamming(nw)
Frame = enframe(data[0], nw, inc, winfunc)
```
## 信号处理代码示例

###生成语音信号
```python
# -*- coding: utf-8 -*-
# 数据读取
import wave
# 数值计算处理
import numpy as np
# 信号处理
import scipy.signal as signal
# 类似matlab的画图
import pylab as pl

# 采样率的设置
framerate = 44100
# 时长
time = 1

# 时间点
t = np.arange(0, time, 1.0/framerate)
# chirp 生成信号函数（频率始终为4kHz）
wave_data = signal.chirp(t, 4000, time, 4000, method='linear') * 1000
wave_data = wave_data.astype(np.short)

# 打开WAV文档
f = wave.open(r"sweep.wav", "wb")

# 配置声道数、量化位数和取样频率
f.setnchannels(1)
f.setsampwidth(2)
f.setframerate(framerate)
# 将wav_data转换为二进制数据写入文件
f.writeframes(wave_data.tostring())
f.close()
```
### 读取语音信号
WAV是Microsoft开发的一种声音文件格式，虽然它支持多种压缩格式，不过它通常被用来保存未压缩的声音数据（PCM脉冲编码调制)。WAV有三个重要的参数：声道数、取样频率和量化位数。

- 声道数：可以是单声道或者是双声道
- 采样频率：一秒内对声音信号的采集次数，常用的有8kHz, 16kHz, 32kHz, 48kHz, 11.025kHz, 22.05kHz, 44.1kHz
- 量化位数：用多少bit表达一次采样所采集的数据，通常有8bit、16bit、24bit和32bit等几种
```python
# -*- coding: utf-8 -*-
import wave
import pylab as pl
import numpy as np

# 打开WAV文档
f = wave.open(r"c:\WINDOWS\Media\ding.wav", "rb")

# 读取格式信息
# (nchannels, sampwidth, framerate, nframes, comptype, compname)
# (声道数, 比特率, 采样率，数组帧数)
# 比特率：用几个字节来表示数据
params = f.getparams()
nchannels, sampwidth, framerate, nframes = params[:4]

# 读取波形数据
str_data = f.readframes(nframes)
f.close()

#将波形数据转换为数组
wave_data = np.fromstring(str_data, dtype=np.short)
# 对于双声道数据做这个处理，上面的数据排列为LRLRLRLR...,需要分开
wave_data.shape = -1, 2
# 数据转置
wave_data = wave_data.T
time = np.arange(0, nframes) * (1.0 / framerate)

# 绘制波形
pl.subplot(211) 
pl.plot(time, wave_data[0])
pl.subplot(212) 
pl.plot(time, wave_data[1], c="g")
pl.xlabel("time (seconds)")
pl.show()
```
### 播放声音
```python
# -*- coding: utf-8 -*-
import pyaudio
import wave

chunk = 1024

wf = wave.open(r"c:\WINDOWS\Media\ding.wav", 'rb')

p = pyaudio.PyAudio()

# 打开声音输出流
stream = p.open(format = p.get_format_from_width(wf.getsampwidth()),
                channels = wf.getnchannels(),
                rate = wf.getframerate(),
                output = True)

# 写声音输出流进行播放
while True:
    data = wf.readframes(chunk)
    if data == "": break
    stream.write(data)

stream.close()
p.terminate()
```
### 录音
```python
# -*- coding: utf-8 -*-
from pyaudio import PyAudio, paInt16 
import numpy as np 
from datetime import datetime 
import wave 


# 将data中的数据保存到名为filename的WAV文件中
def save_wave_file(filename, data): 
    wf = wave.open(filename, 'wb') 
    wf.setnchannels(1) 
    wf.setsampwidth(2) 
    wf.setframerate(SAMPLING_RATE) 
    wf.writeframes("".join(data)) 
    wf.close() 



NUM_SAMPLES = 2000      # pyAudio内部缓存的块的大小
SAMPLING_RATE = 8000    # 取样频率
LEVEL = 1500            # 声音保存的阈值
COUNT_NUM = 20          # NUM_SAMPLES个取样之内出现COUNT_NUM个大于LEVEL的取样则记录声音
SAVE_LENGTH = 8         # 声音记录的最小长度：SAVE_LENGTH * NUM_SAMPLES 个取样

# 开启声音输入
pa = PyAudio() 
stream = pa.open(format=paInt16, channels=1, rate=SAMPLING_RATE, input=True, 
                frames_per_buffer=NUM_SAMPLES) 

save_count = 0 
save_buffer = [] 

while True: 
    # 读入NUM_SAMPLES个取样
    string_audio_data = stream.read(NUM_SAMPLES) 
    # 将读入的数据转换为数组
    audio_data = np.fromstring(string_audio_data, dtype=np.short) 
    # 计算大于LEVEL的取样的个数
    large_sample_count = np.sum( audio_data > LEVEL ) 
    print np.max(audio_data) 
    # 如果个数大于COUNT_NUM，则至少保存SAVE_LENGTH个块
    if large_sample_count > COUNT_NUM: 
        save_count = SAVE_LENGTH 
    else: 
        save_count -= 1 

    if save_count < 0: 
        save_count = 0 

    if save_count > 0: 
        # 将要保存的数据存放到save_buffer中
        save_buffer.append( string_audio_data ) 
    else: 
        # 将save_buffer中的数据写入WAV文件，WAV文件的文件名是保存的时刻
        if len(save_buffer) > 0: 
            filename = datetime.now().strftime("%Y-%m-%d_%H_%M_%S") + ".wav" 
            save_wave_file(filename, save_buffer) 
            save_buffer = [] 
            print filename, "saved" 
```

## 信号绘图

###时域信号图
```python
pl.plot(t, wave_data)
pl.xlabel("time (seconds)")
pl.show()
```

### 频域信号图
```python
# 采样点数，修改采样点数和起始位置进行不同位置和长度的音频波形分析
N=44100
start=0 #开始采样位置
df = framerate/(N-1) # 分辨率
freq = [df*n for n in range(0, N)] #N个元素
wave_data2 = wave_data[start:start+N]
# 正确的幅值
c=np.fft.fft(wave_data2)*2/N
#常规显示采样频率一半的频谱
d=int(len(c)/2)
#仅显示频率在4000以下的频谱
pl.plot(freq[:d-1], abs(c[:d-1]), 'r')
pl.show()
```

### 语谱图
```python
pl.specgram(wave_data, Fs=framerate, scale_by_freq=True, sides='default')
pl.ylabel('Frequency(Hz)')
pl.xlabel('Time(s)')
pl.show()
```