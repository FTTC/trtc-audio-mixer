# 背景音乐和音效实现方案

## 支持的平台

| 操作系统 |      浏览器类型       | 浏览器最低版本要求 | 增加背景音 | 设置音量大小 | 设置播放速率 | 设置播放位置 | 原音频轨道增加背景音发布后，进行多次替换音频轨道操作 |
| :------: | :-------------------: | :----------------- | ---------- | ------------ | ------------ | ------------ | -------------------------------------------- |
|  Mac OS  | 桌面版 Chrome 浏览器  | 56+                | ✔          | ✔            | ✔            | ✔            | ✔                                            |
|  Mac OS  | 桌面版 Safari 浏览器  | 11+                | ✔          | ✖            | ✖            | ✖            | ✖                                            |
|  Mac OS  | 桌面版 Firefox 浏览器 | 56+                | ✔          | ✔            | ✖            | ✔            | ✔                                            |
|  Mac OS  |      桌面版Edge       | 80+                | ✔          | ✔            | ✔            | ✔            | ✔                                            |
| Windows  | 桌面版 Chrome 浏览器  | 56+                | ✔          | ✔            | ✔            | ✔            | ✔                                            |
| Windows  |   桌面版 QQ 浏览器（极速模式）    | 10.4+               | ✔          | ✔            | ✔            | ✔            | ✔                                            |
| Windows  | 桌面版 Firefox 浏览器 | 56+                | ✔          | ✔            | ✖            | ✔            | ✔                                            |
| Windows  |      桌面版Edge       | 80+                | ✔          | ✔            | ✔            | ✔            | ✔                                            |
|   iOS    | 移动版 Safari 浏览器  | 14+                | ✔          | ✖            | ✖            | ✖            | ✖                                            |
|   iOS    |     微信内嵌网页      | ✖                  |            |              |              |              |                                              |
| Android  | 移动版 Chrome 浏览器  | 81+                  | ✔          | ✔            | ✔            | ✔            | ✔                                             |
| Android  |     微信内嵌网页(TBS内核)      |  ✔               |    ✔        |    ✔           |   ✔          |   ✔          | ✖                                            |
| Android  |   移动版 QQ 浏览器    | ✖                 |            |              |              |              |                                              |

**注意事项**
- 内嵌 WebView 的应用和 Chrome 浏览器对增加背景音乐功能的支持度，都与设备对 Web Audio API 的支持度有关。

## 功能描述

在通话或者直播过程中，除了用户自己说话的声音，有时候需要播放背景音乐或者音效，并且让频道内的其他人也听到，比如需要给游戏添加音效，或者需要播放背景音乐等。

本文介绍如何使用 `AudioMixerPlugin` 插件来实现在如何在 [TRTC](https://www.npmjs.com/package/trtc-js-sdk) 中实现播放背景音乐的功能。

[点击此处](https://trtc-1252463788.cos.ap-guangzhou.myqcloud.com/web/demo/audio-mixer/index.html) 查看在线 demo 进行体验。

## 对接攻略

### Step1. 创建音乐实例

`AudioMixerPlugin` 插件需要和 `TRTC` 在同一作用域引入。

```javascript
import TRTC from 'trtc-js-sdk';
import AudioMixerPlugin from 'trtc-audio-mixer';
```

调用 `AudioMixerPlugin.createAudioSource(params)` 方法创建一个 `AudioSource` 的音乐实例，参数如下：

- `url`：String类型，必选，音乐文件的地址，可使用线上文件地址，也可以使用本地选择对象生成的 [blob](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob) URL。
- `loop`: Boolean类型，可选，是否循环播放，默认 `false` 不循环。
- `volume`：Number类型，可选，音量设置（0 - 1）， 默认为1。

```javascript
// 通过线上文件地址，也可以通过选择本地文件生成 blob url 加载，参考文末的 API 说明
let audioSourceA = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceA.mp3' });

```

**注意事项**

- 播放在线音效文件时，您需要配置线上音乐文件的 [CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS) ，线上文件必须为 `https` 协议。
- 支持的格式为 MP3，AAC（以及浏览器支持的其他音频格式）。
- 网页在没有用户交互之前，浏览器禁止网页播放带有声音的媒体，建议引导用户执行点击行为后进行相关操作， 参考：[Autoplay_guide](https://developer.mozilla.org/en-US/docs/Web/Media/Autoplay_guide) 。

### Step2. 发布背景音乐

调用 `AudioMixerPlugin.mix(params)` 方法，获取已加入背景音乐的 mixedAudioTrack，可用于替换发布流的音频轨道：

- `targetTrack`: 可选，需要加入背景音乐的音频轨道。当不传递的时候则生成只有背景音的轨道。
- `sourceList`: 必选，Array类型，传入第一步创建的音乐实例，例如： `[ audioSourceA, audioSourceB ]`。

#### 场景一：发布前替换

在 `localStream` 发布之前，将 `mixedAudioTrack` 替换掉 `localStream` 里的音频轨道。

```javascript
let audioSourceA = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceA.mp3' });
let audioSourceB = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceB.mp3' });

const localStream = TRTC.createStream({ userId: 'user', audio: true, video: true });
localStream
	.initialize()
	.then(() => {
		// 1. 获取麦克风轨道
		let originAudioTrack = localStream.getAudioTrack();
	
		// 2. 与 audioSourceA audioSourceB 混合后生成 mixedAudioTrack
		mixedAudioTrack = AudioMixerPlugin.mix({targetTrack: originAudioTrack, sourceList: [ audioSourceA, audioSourceB ]});
		
		// 3. 替换麦克风轨道
		return localStream.replaceTrack(mixedAudioTrack);
	})
	.then(() => {
        // 4. 发布
        client.publish(localStream);

        // 5. 调用 play 方法播放，这时通话双方都能听到背景音乐
        audioSourceA.play();
    });
```

#### 场景二：发布后替换

```javascript
let audioSourceA = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceA.mp3' });
let audioSourceB = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceB.mp3' });

// 1. 获取麦克风轨道
let originAudioTrack = localStream.getAudioTrack();

// 2. 将麦克风轨道 与 audioSourceA audioSourceB 进行混合
mixedAudioTrack = AudioMixerPlugin.mix({ targetTrack: originAudioTrack, sourceList: [ audioSourceA, audioSourceB ] });

// 3. 替换 localStream 的音频轨道
localStream.replaceTrack(mixedAudioTrack);

// 4. 调用 play 方法播放，这时通话双方都能听到背景音乐
audioSourceA.play();
audioSourceB.play();

```

#### 场景三：静音麦克风，播放背景音

```javascript
let audioSourceA = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceA.mp3' });
let audioSourceB = AudioMixerPlugin.createAudioSource({ url: 'https://audioSourceB.mp3' });

// 暂存包含了麦克风和背景音的轨道，可用于取消静音麦克风
let microAndBgmTrack = localStream.getAudioTrack();

// 1. 生成只有 audioSourceA audioSourceB 的背景音
let bgmTrack = AudioMixerPlugin.mix({sourceList: [audioSourceA, audioSourceB]});

// 2. 替换 localStream 的 audio track，此时只有背景音，没有麦克风声音
localStream.replaceTrack(bgmTrack);

// 如果想开启麦克风，保留背景音，只需要恢复包含了麦克风和背景音的轨道即可
localStream.replaceTrack(microAndBgmTrack);

```

## API 说明

### AudioMixerPlugin

用于实现背景音乐、音效的 AudioMixerPlugin 插件，可以通过 TRTC.AudioMixerPlugin 访问。

- 检测当前浏览器环境是否拥有给音频轨道增加背景音乐的能力 isSupported()
- 创建音乐实例 createAudioMixer(params)
- 生成有背景音乐的音频轨道 mix(params)

### isSupported()

用于检测当前浏览器是否支持添加背景音的功能。返回值为 Boolean 类型，true 为支持，false 为不支持。

```javascript
const result = AudioMixerPlugin.isSupported();
if (!result) {
  alert('Your browser is not compatible with AudioMixerPlugin');
}
```

### createAudioSource(params)

用于创建一个 `AudioSource` 的音乐实例用于给音频轨道增加背景音乐。

**Example:**

例一：通过文件地址创建

```javascript
// 线上文件地址
let audioSourceA = AudioMixerPlugin.createAudioSource({ url: 'https://XXXX.mp3', loop: true, volume: 0.5 });

// 文件相对路径
let audioSourceB = AudioMixerPlugin.createAudioSource({ url: '../XXXX.mp3' });

audioSourceA.play();
```

例二：通过选择本地文件生成 blob url 加载

```javascript
// 通过 `<input type="file" id="fileInput">` 选择本地文件生成 blob url 加载
fileInput.addEventListener('change', function() {
  var file = fileInput.files[0];

  var reader = new FileReader();
  reader.onload = function(e) {
    let result = e.target.result;
    let audioSourceC = AudioMixerPlugin.createAudioSource({ url: result, loop: true, volume: 0.5 });
  };
  reader.readAsDataURL(file);
});
```

#### Params:

| Name   | Type      | Attributes   | Description               |
| ------ | --------- | ------------ | ------------------------- |
| url    | `string`  |              | 音乐文件地址              |
| loop   | `boolean` | `<optional>` | 是否循环播放，默认为false |
| volume | `number`  | `<optional>` | 设置初始音量，默认为1     |


#### AudioSource 有如下方法：

| Name                    | Type                                     | Description                                  |
| ----------------------- | ---------------------------------------- | -------------------------------------------- |
| play()                  |                                          | 播放音乐                                     |
| pause()                 |                                          | 暂停音乐                                     |
| resume()                |                                          | 重置音乐                                     |
| stop()                  |                                          | 停止音乐                                     |
| duration()              |                                          | 获取音乐的时长（s）                           |
| setPosition(time)       | time: `number`（秒）                     | 设置音乐位置（不超过音乐的最长时长）             |  
| getPosition()           |                                          | 获取音乐当前播放位置                          |
| setVolume(volume)       | volume: `number` (0 - 1)                 | 设置音量大小 （同时影响本地和远端播放的音量）    |
| getVolume()             |                                          | 获取音量大小                                 |
| setPlayBackRate(rate)   | rate: `number` (1 - 5)                   | 设置播放速率 （1 - 5）                       |
| getPlayBackRate()       |                                          | 获取播放速率                                 |
| loop(loop)              | loop: `boolean`                          | 设置是否循环播放，不传 loop 返回 loop 的状态    |
| on(eventName, handler)  | eventName: `number`  handler: `function` | 监听事件                                     |
| off(eventName, handler) | eventName: `number`  handler: `function` | 取消监听事件                                 |

**事件列表**

可以监听的事件与 HTMLAudioElement 的标准事件相同，查看：[事件列表](https://developer.mozilla.org/zh-CN/docs/Web/Guide/Events/Media_events) 。

| Name     | Description  |
| -------- | ------------ |
| play     | 开始播放音乐  |
| progress | 音乐加载      |
| end      | 音乐播放结束  |
|        ......           |
| error    | audio 加载、播放异常等错误，参考文章的常见问题部分  |

```javascript
// 监听播放事件
audioSourceA.on('play', function play(event) {
  console.log('play event trigger', event);
});

// 取消监听所有事件
audioSourceA.off('*', function cancel(event)  {
  console.log('取消监听所有事件', event);
});
```

### mix(params)

用于将创建的音乐实例与音频轨道进行混音，targetTrack 为待混合的音频轨道，sourceList 为 createAudioSource 创建的音乐实例数组, 例：[ audioSourceA, audioSourceB ] 。

当 targetTrack 不传参的时候， 此时生成纯背景音的音轨。

**Example:**

targetTrack 和 sourceList 都传参

```javascript
// 1. 获取麦克风轨道
let originAudioTrack = localStream.getAudioTrack();

// 2. 将麦克风轨道与 audioSourceA audioSourceB 进行混合
mixedAudioTrack = AudioMixerPlugin.mix({ targetTrack: originAudioTrack, sourceList: [ audioSourceA, audioSourceB ] });

// 3. 替换麦克风轨道即可
localStream.replaceTrack(mixedAudioTrack);
```

只传 sourceList 数组

```javascript
// 生成只有 audioSourceA audioSourceB 的背景音
let bgmTrack = AudioMixerPlugin.mix({ sourceList: [ audioSourceA, audioSourceB ] });

```

### Params:

| Name        | Type                 | Attributes   | Description            |
| ----------- | -------------------- | ------------ | ---------------------- |
| targetTrack | `MediaStreamTrack`   | `<optional>` | 目标音频轨道             |
| sourceList  | `Array<AudioSource>` |              | AudioSource 的数组对象  |


## 常见问题

**1. 出现跨域错误**

例如：Access to audio at XXX from origin XXX has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.

解决：您需要配置线上音乐文件的 CORS 协议。

**2. 音频格式不对，浏览器无法播放**

例如：NotSupportedError: The operation is not supported.

解决：您需要使用浏览器支持的音频格式。
