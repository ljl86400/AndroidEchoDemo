## 使用 android 自带回声消除

Android 在4.1 API leve 16 添加了回声消除处理 **AcousticEchoCanceler**

不过由于手机厂商太多，碎片化严重，实现效果也不一样，这里只做测试性使用

记录一下步骤：

**检测系统是否支持回声消除**

**创建录制，并拿到sessionid**

**创建回声消除 关联录制的sessionid**

**创建播放，关联录制的sessionid**

这三者由录制的sessionid 关联起来了，就能够实处理掉mic采集到的由喇叭播放出的声音了。

看代码：
```
    public static boolean chkNewDev()
    {
          return android.os.Build.VERSION.SDK_INT >= 16;
    }
    
	public static boolean isDeviceSupport()
	{
        return AcousticEchoCanceler.isAvailable();
	}
```
检测是否支持，居然存在不准确的情况？
```
int InitAudioRecord()
		{
			m_bufferSizeInBytes = AudioRecord.getMinBufferSize(m_sampleRateInHz,  m_channelConfig, m_audioFormat);  	
			if (chkNewDev())
			{
				m_audioRecord = new AudioRecord(MediaRecorder.AudioSource.VOICE_COMMUNICATION, m_sampleRateInHz, m_channelConfig, m_audioFormat, m_bufferSizeInBytes);
			}
			else
			{
				m_audioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC, m_sampleRateInHz, m_channelConfig, m_audioFormat, m_bufferSizeInBytes);
			}
			
			int packSize = Math.min(960, m_bufferSizeInBytes*2);
			audioData = new short[packSize];
			return 0;
		}
```
如果系统支持回声消除，则用**MediaRecorder.AudioSource.VOICE_COMMUNICATION**
```
public int GetSessionId()
		{
			return m_audioRecord.getAudioSessionId();
		}
```
获取mic的sessionid，给回声消除和播放关联用
```
public boolean initAEC(int audioSession)
	{
		if (m_canceler != null)
		{
			return false;
		}
		m_canceler = AcousticEchoCanceler.create(audioSession);
		m_canceler.setEnabled(true);
		return m_canceler.getEnabled();
	}
```
创建回声消除 并使能
```
		int InitAudioTrack()
		{
			int m_sampleRateInHz = 48000;  
			int m_channelConfig  =  AudioFormat.CHANNEL_IN_STEREO;   
			int m_audioFormat    = AudioFormat.ENCODING_PCM_16BIT; 
			int m_bufferSizeInBytes = 1024*4;
			
			if (chkNewDev() && m_recorder != null)
			{
				m_audioTrack = new AudioTrack(AudioManager.STREAM_VOICE_CALL, m_sampleRateInHz, m_channelConfig, m_audioFormat, m_bufferSizeInBytes, AudioTrack.MODE_STREAM, m_recorder.GetSessionId());
			}
			else
			{
				m_audioTrack = new AudioTrack(AudioManager.STREAM_VOICE_CALL, m_sampleRateInHz, m_channelConfig, m_audioFormat, m_bufferSizeInBytes, AudioTrack.MODE_STREAM);
			}
			return 0;
		}
```
如果系统支持回声处理 则创建关联mic 的sessionid

ok，走通了

```
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
        AudioManager audioManager = (AudioManager) this.getSystemService(Context.AUDIO_SERVICE);
        audioManager.setMode(AudioManager.MODE_IN_COMMUNICATION);
        audioManager.setSpeakerphoneOn(true);
```

忘了说，在程序启动的时候需要设置一下 **AudioManager.MODE_IN_COMMUNICATION** 和 外放来验证回声处理效果

验证步骤：
1】打开录音机功能并开始录音，对着麦克风说：音频录制开始；
2】将录音机功能切换至后台，并打开一个音乐播放软件播放音乐，对着麦克风说：音乐播放功能已经打开
3】将音乐播放软件切换至后台，并打开回声消除App，进入录音机App界面，对着麦克风说：回声消除功能已经打开
4】切换到回声消除App界面，退出此App，切回录音机界面，对着麦克风说：回声消除功能已经关闭
5】退出音乐播放软件，对着麦克风说：音乐播放功能已经关闭
6】停止录音，并切换至录音播放功能，对比回升消除功能的效果。

重新实验时候，需要退出录音机App，因为已经打开的App添加了回声消除功能，此功能会一直存在，不会随着“回声消除App”的退出而关闭，要想关闭此功能，必须将已经存在的录音机进程关闭，并重新建立一个新的录音机进程。
