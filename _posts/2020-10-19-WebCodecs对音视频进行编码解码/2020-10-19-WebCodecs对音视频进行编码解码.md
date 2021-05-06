---
layout: post
title: WebCodecs对音视频进行编码解码
tags: ['2020']
comments: true
---

# WebCodecs

- 草案：https://wicg.github.io/web-codecs/
- Github：https://github.com/WICG/web-codecs

允许 Web 应用程序对音频和视频进行编码和解码的 API


# 在 Chrome >= 86 的版本进行体验

- Chrome地址栏输入：chrome://flags/#enable-experimental-web-platform-features，设置成 `Enabled`
- 通过命令行启用 Chrome：`/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --enable-blink-features=WebCodecs`

```js
// 通过 VideoEncoder API 检查当前浏览器是否支持
if ('VideoEncoder' in window) {
  // 支持 WebCodecs API
}
```

# Web 编解码器 API 出现的原因

现在已经有很多 Web API 进行媒体操作： [Media Stream API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API), [Media Recording API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API), [Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API)、[WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)，但是没有提供一些底层 API 给到 Web 开发者进行帧操作或者对已经编码的视频进行解封装操作。

很多音视频编辑器为了解决这个问题，使用了 WebAssembly 把[音视频编解码](https://en.wikipedia.org/wiki/Video_codec)带到了浏览器，但是有个问题是现在的浏览器很多已经在底层支持了音视频编解码，并且还进行了很多硬件加速的调优，如果使用 WebAssembly 重新打包这些能力，似乎浪费人力和计算机资源。

所以就诞生了 [WebCodecs API](https://wicg.github.io/web-codecs/)，暴露媒体 API 来使用浏览器已经有的一些能力，例如：

- 视频和音频解码
- 视频和音频编码
- 原始视频帧
- 图像解码器

这时候一些 web 媒体开发项目例如视频编辑器、视频会议、视频流的操作就方便多了

项目进展查看：https://github.com/WICG/web-codecs

# WebCodecs 处理流程

`frames` 是视频处理的核心，因此 WebCodecs 大多数类要么消耗 `frames` 要么生产 `frames`。Video encoders 把 `frames` 转换为 `encoded chunks`，Video Decoders 把 `encoded chunks` 转换为 `frames`。这一切都在非主线程异步处理，所以可以保证主线程速度。

当前，在 WebCodecs 中，在页面上显示 `frame` 的唯一方法是将其转换为 [ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap) 并在 canvas 上绘制位图或将其转换为 [WebGLTexture](https://developer.mozilla.org/en-US/docs/Web/API/WebGLTexture)

# Webcodecs 实际应用

## Encoding

现在有两种方法把已经存在的图片转换为 `VideoFrame`

- 一种是通过 [ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap) 创建 frame
- 第二种是通过 `VideoTrackReader` 设置方法来处理从 `[MediaStreamTrack](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack)` 产生的 frame，当需要从摄像机或屏幕捕获视频流时，这个 API 很有用

### ImageBitmap

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210429091535_d1c49c49839de8210cf326cc6d8ef22c.png)

```js
let cnv = document.createElement('canvas');
// draw something on the canvas
...
let bitmap = await createImageBitmap(cnv);
let frameFromBitmap = new VideoFrame(bitmap, { timestamp: 0 });
```

第一种是直接从 ImageBitmap 创建 frame。只需调用 `new VideoFrame()` 构造函数并为其提供 bitmap 和展示时间戳

### VideoTrackReader

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210429091550_382ee962dab29bef0dd8d5699e036c95.png)

```js
const framesFromStream = [];
const stream = await navigator.mediaDevices.getUserMedia({ … });
const vtr = new VideoTrackReader(stream.getVideoTracks()[0]);
vtr.start((frame) => {
  framesFromStream.push(frame);
});
```

使用 VideoEncoder 将 frame 编码为 EncodedVideoChunk 对象，VideoEncoder 需要两个对象：

- 带有 `output` 和 `error` 两个方法的初始化对象，传递给 `VideoEncoder` 后无法修改
- Encoder 配置对象，其中包含输出视频流的参数。可以稍后通过调用 `configure()` 来更改这些参数

```js
const init = {
  output: handleChunk,
  error: (e) => {
    console.log(e.message);
  }
};

let config = {
  codec: 'vp8',
  width: 640,
  height: 480,
  bitrate: 8_000_000, // 8 Mbps
  framerate: 30,
};

let encoder = new VideoEncoder(init);
encoder.configure(config);
```

设置好 encoder 后，可以接受 frames 了，当开始从 media stream 接受 frames 的时候，传递给 `VideoTrackReader.start()` 的 callback 就会被执行，把 frame 传递给 encoder，需要定时检查 frame 防止过多的 frames 导致处理问题。注意：`encoder.configure()` 和 `encoder.encode()` 会立即返回，不会等待真正处理完成。如果处理完成 `output()`方法会被调用，入参是 `encoded chunk`。再注意：`encoer.encode()` 会消耗掉 frame，如果 frame 需要后面使用，需要调用 `clone` 来复制它。

```js
let frameCounter = 0;
let pendingOutputs = 0;
const vtr = new VideoTrackReader(stream.getVideoTracks()[0]);

vtr.start((frame) => {
  if (pendingOutputs > 30) {
    // 有太多帧正在处理中，编码器承受不过来，不添加新的处理帧了
    return;
  }
  frameCounter++;
  pendingOutputs++;
  const insert_keyframe = frameCounter % 150 === 0;
  encoder.encode(frame, { keyFrame: insert_keyframe });
});
```

最后就是完成 handleChunk 方法，通常，此功能是通过网络发送数据块或将它们封装到到媒体容器中

```js
function handleChunk(chunk) {
  let data = new Uint8Array(chunk.data);  // actual bytes of encoded data
  let timestamp = chunk.timestamp;        // media time in microseconds
  let is_key = chunk.type == 'key';       // can also be 'delta'
  pending_outputs--;
  fetch(`/upload_chunk?timestamp=${timestamp}&type=${chunk.type}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/octet-stream' },
    body: data
  });
}
```

有时候需要确保所有 pending 的 encoding 请求完成，调用 `flush()`

```js
await encoder.flush();
```

## Decoding

设置 `VideoDecoder` 和上面类似，需要传递 `init` 和 `config` 两个对象

```js
const init = {
  output: handleFrame,
  error: (e) => {
    console.log(e.message);
  },
};

const config = {
  codec: "vp8",
  codedWidth: 640,
  codedHeight: 480,
};

const decoder = new VideoDecoder(init);
decoder.configure(config);
```

设置好 `decoder` 之后就可以给它喂 `EncodedVideoChunk` 对象了，通过 `[BufferSouce](https://developer.mozilla.org/en-US/docs/Web/API/BufferSource)` 来创建 `chunk`

```js
const responses = await downloadVideoChunksFromServer(timestamp);
for (let i = 0; i < responses.length; i++) {
  const chunk = new EncodedVideoChunk({
    timestamp: responses[i].timestamp,
    data: new Uint8Array(responses[i].body),
  });
  decoder.decode(chunk);
}
await decoder.flush();

```

![03.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210429091610_1495b150a15d1fcdc5becf9f90cbbdb9.png)

渲染 `decoded frame` 到页面上分为三步：

- 把 `VideoFrame` 转换为 `[ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap)`
- 等待合适的时机显示 frame
- 将 image 画到 canvas 上

当一个 frame 不再需要的时候，调用 `destroy()` 在垃圾回收之前手动销毁他，这可以减少页面内存占用

```js
const cnv = document.getElementById("canvas_to_render");
const ctx = cnv.getContext("2d", { alpha: false });
const readyFrames = [];
let underflow = true;
let timeBase = 0;

function handleFrame(frame) {
  readyFrames.push(frame);
  if (underflow) setTimeout(renderFrame, 0);
}

function delay(time_ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, time_ms);
  });
}

function calculateTimeTillNextFrame(timestamp) {
  if (timeBase == 0) timeBase = performance.now();
  const media_time = performance.now() - timeBase;
  return Math.max(0, timestamp / 1000 - media_time);
}

async function renderFrame() {
  if (readyFrames.length === 0) {
    underflow = true;
    return;
  }
  const frame = readyFrames.shift();
  underflow = false;

  const bitmap = await frame.createImageBitmap();
  frame.destroy();
  // 根据帧的时间戳，计算在显示下一帧之前需要的实时等待时间
  const timeTillNextFrame = calculateTimeTillNextFrame(frame.timestamp);
  await delay(timeTillNextFrame);
  ctx.drawImage(bitmap, 0, 0);

  // 立即下一帧渲染
  setTimeout(renderFrame, 0);
}
```

## Demo

体验地址：https://ringcrl.github.io/static/webcodecs-demo/

如果打不开参考上面【在 Chrome >= 86 的版本进行体验】

```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="UTF-8">
  <style>
    canvas {
      padding: 10px;
      background: gold;
    }

    button {
      background-color: #555555;
      border: none;
      color: white;
      padding: 15px 32px;
      width: 150px;
      text-align: center;
      display: block;
      font-size: 16px;
    }
  </style>
</head>

<body>
  <button onclick="playPause()">Pause</button>
  <canvas id="dst" width="640" height="480"></canvas>
  <canvas style="visibility: hidden;" id="src" width="640" height="480"></canvas>
  <script src="./main.js"></script>

</body>

</html>
```

```js
const codecString = "vp8";
let keepGoing = true;

function playPause() {
  keepGoing = !keepGoing;
  const btn = document.querySelector("button");
  if (keepGoing) {
    btn.innerText = "Pause";
  } else {
    btn.innerText = "Play";
  }
}

function delay(time_ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, time_ms);
  });
}

async function startDrawing() {
  const cnv = document.getElementById("src");
  const ctx = cnv.getContext("2d", { alpha: false });

  ctx.fillStyle = "white";
  const { width } = cnv;
  const { height } = cnv;
  const cx = width / 2;
  const cy = height / 2;
  const r = Math.min(width, height) / 5;
  const drawOneFrame = function drawOneFrame(time) {
    const angle = Math.PI * 2 * (time / 5000);
    const scale = 1 + 0.3 * Math.sin(Math.PI * 2 * (time / 7000));
    ctx.save();
    ctx.fillRect(0, 0, width, height);

    ctx.translate(cx, cy);
    ctx.rotate(angle);
    ctx.scale(scale, scale);

    ctx.font = "30px Verdana";
    ctx.fillStyle = "black";
    const text = "😊📹📷Hello WebCodecs 🎥🎞️😊";
    const size = ctx.measureText(text).width;
    ctx.fillText(text, -size / 2, 0);
    ctx.restore();
    window.requestAnimationFrame(drawOneFrame);
  };
  window.requestAnimationFrame(drawOneFrame);
}

function captureAndEncode(processChunk) {
  const cnv = document.getElementById("src");
  const fps = 60;
  let pendingOutputs = 0;
  let frameCounter = 0;
  const stream = cnv.captureStream(fps);
  const vtr = new VideoTrackReader(stream.getVideoTracks()[0]);

  const init = {
    output: (chunk) => {
      pendingOutputs--;
      processChunk(chunk);
    },
    error: (e) => {
      console.log(e.message);
      vtr.stop();
    },
  };

  const config = {
    codec: codecString,
    width: cnv.width,
    height: cnv.height,
    bitrate: 10e6,
    framerate: fps,
  };

  const encoder = new VideoEncoder(init);
  encoder.configure(config);

  vtr.start((frame) => {
    if (!keepGoing) return;
    if (pendingOutputs > 30) {
      // Too many frames in flight, encoder is overwhelmed
      // let's drop this frame.
      return;
    }
    frameCounter++;
    pendingOutputs++;
    const insert_keyframe = frameCounter % 150 === 0;
    encoder.encode(frame, { keyFrame: insert_keyframe });
  });
}

async function frameToBitmapInTime(frame, timeout_ms) {
  const options = { colorSpaceConversion: "none" };
  const convertPromise = frame.createImageBitmap(options);

  if (timeout_ms <= 0) return convertPromise;

  const results = await Promise.all([convertPromise, delay(timeout_ms)]);
  return results[0];
}

function startDecodingAndRendering() {
  const cnv = document.getElementById("dst");
  const ctx = cnv.getContext("2d", { alpha: false });
  const readyFrames = [];
  let underflow = true;
  let timeBase = 0;

  function calculateTimeTillNextFrame(timestamp) {
    if (timeBase === 0) timeBase = performance.now();
    const mediaTime = performance.now() - timeBase;
    return Math.max(0, timestamp / 1000 - mediaTime);
  }

  async function renderFrame() {
    if (readyFrames.length === 0) {
      underflow = true;
      return;
    }
    const frame = readyFrames.shift();
    underflow = false;

    const bitmap = await frame.createImageBitmap();
    // 根据帧的时间戳，计算在显示下一帧之前需要的实时等待时间
    const timeTillNextFrame = calculateTimeTillNextFrame(frame.timestamp);
    await delay(timeTillNextFrame);
    ctx.drawImage(bitmap, 0, 0);

    // 立即下一帧渲染
    setTimeout(renderFrame, 0);
    frame.destroy();
  }

  function handleFrame(frame) {
    readyFrames.push(frame);
    if (underflow) {
      underflow = false;
      setTimeout(renderFrame, 0);
    }
  }

  const init = {
    output: handleFrame,
    error: (e) => {
      console.log(e.message);
    },
  };

  const config = {
    codec: codecString,
    codedWidth: cnv.width,
    codedHeight: cnv.height,
  };

  const decoder = new VideoDecoder(init);
  decoder.configure(config);
  return decoder;
}

function main() {
  if (!("VideoEncoder" in window)) {
    document.body.innerHTML = "<h1>WebCodecs API is not supported.</h1>";
    return;
  }
  startDrawing();
  const decoder = startDecodingAndRendering();
  captureAndEncode((chunk) => {
    decoder.decode(chunk);
  });
}

document.body.onload = main;
```

# 参考地址

[Video processing with WebCodecs](https://web.dev/webcodecs/)
