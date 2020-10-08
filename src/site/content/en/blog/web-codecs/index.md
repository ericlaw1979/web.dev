---
title: Video processing with WebCodecs
subhead: Manipulating video stream components
description: |
  Work with components of a video stream, such as frames and unmuxed chunks of encoded video or audio.
date: 2020-10-13
updated: 2020-10-13
authors:
 - djuffin
origin_trial:
    url: https://developers.chrome.com/origintrials/#/view_trial/-7811493553674125311
tags:
  - blog
  - media
  - video
feedback:
  - api
---

Modern web technologies provide ample ways to work with video.
[Media Stream API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API),
[Media Recording API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API)
,
[Media Source API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API),
 and [WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) add up
to a rich tool set for recording, transferring,  and playing video streams.
While solving these high-level tasks, the existing APIs don't let web
programmers work with individual components of a video stream, such as frames
and unmuxed chunks of encoded video or audio.
To get low level access to these basic components, developers have been using
web assembly to bring [video and audio
codecs](https://en.wikipedia.org/wiki/Video_codec) into the browser. But given
that modern browsers already ship with a variety of codecs (which are often
accelerated by hardware), repackaging them as web assembly seems like a waste of
human and computer resources.

[WebCodecs API](https://wicg.github.io/web-codecs/) eliminates this inefficiency
by giving programmers a way to use media components that are already present in
the browser:

+   Video and audio decoders
+   Video and audio encoders
+   Raw video frames
+   Image decoders

WebCodecs API is useful for web applications that require full control over the
way media content is processed, such as video editors, video conferencing, video
streaming etc.

## Current status {: #status }

<div class="w-table-wrapper">

| Step                                         | Status                       |
| -------------------------------------------- | ---------------------------- |
| 1. Create explainer                          | [Complete](https://github.com/WICG/web-codecs/blob/master/explainer.md)        |
| 2. Create initial draft of specification     | [Complete](https://wicg.github.io/web-codecs/)         |
| **3. Gather feedback & iterate on design**   | [**In Progress**](#feedback) |
| **4. Origin trial**                          | [**In Progress**](#ot)       |
| 5. Launch                                    | Not started                  |

</div>

## Video processing workflow

Frames are the centerpiece in video processing, thus in WebCodecs most classes
either consume or produce frames. Video encoders convert frames into encoded
chunks. Video decoders do the opposite. Track readers turn video tracks into a
sequence of frames. By design all these transformations happen asynchronously.
WebCodecs API tries to keep the web responsive by keeping the heavy lifting of
video processing off the main thread.
Currently in WebCodecs the only way to show a frame on the page is to convert it
into an
[`ImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap)
and either draw the bitmap on a canvas or convert it into a `WebGL` texture.



![image](insert_image_url_here)

## WebCodecs in action

### Encoding

It all starts with a `VideoFrame`.  There are two ways to convert existing
pictures into `VideoFrame` objects.

The first is to create a frame directly from an
[ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap).
Just call the `VideoFrame()` constructor and give it a bitmap and a presentation
timestamp.

```js
let cnv = document.createElement("canvas");
// draw something on the canvas
...
let bitmap = await createImageBitmap(cnv);
let frame_from_bitmap = new VideoFrame(bitmap, { timestamp: 0 });
```

The second is to use `VideoTrackReader` to set a function that will be called
each time a new frame appears in a
[`MediaStreamTrack`](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack).
This is useful when you need to capture a video stream from a camera or the
screen.

```js
let frames_from_stream = [];
let stream = await navigator.mediaDevices.getUserMedia({ ... });
let vtr = new VideoTrackReader(stream.getVideoTracks()[0]);
vtr.start((frame) => {
  frames_from_stream.push(frame);
});
```

No matter where they are coming from, frames can be encoded into
`EncodedVideoChunk` objects with a `VideoEncoder`.

Before encoding `VideoEncoder` needs to be given two javascript objects:

+   Init dictionary with two functions for handling encoded chunks and
    errors. These functions are developer-defined and can't be changed after
    they're passed to the `VideoEncoder` constructor.
+   Encoder configuration object, which contains parameters for the output
    video stream. You can change these parameters later  by calling `configure()`.

```js
const init = {
  output: handleChunk,
  error: (e) => {
    console.log(e.message);
  }
};

let config = {
  codec: "vp8",
  width: 640,
  height: 480,
  bitrate: 8_000_000,     // 8 Mbps
  framerate: 30,
};

let encoder = new VideoEncoder(init);
encoder.configure(config);
```

After the encoder has been set up,  it's ready to start accepting frames. When
frames are coming from a media stream, `VideoTrackReader` will pump frames into
the encoder, periodically inserting
[keyframes](https://en.wikipedia.org/wiki/Key_frame#Video_compression) and
checking that the encoder is not overwhelmed with incoming frames.
Both `configure()` and `encode()` return immediately without waiting for the
actual work to complete. It allows several frames to queue for encoding at the
same time. But it makes error reporting somewhat cumbersome, errors are reported
either by immediately throwing exceptions or by calling the `error()`
callback. Some errors are easy to detect immediately, others become evident
only during encoding. If encoding completes successfully the `output()`
callback is called with a new encoded chunk as an argument.
Another important detail here is that `encode()` consumes the frame, if the
frame is needed later (for example, to encode with another encoder) it needs to
be duplicated by calling `clone()`.

```js
let frame_counter = 0;
let pending_outputs = 0;
let vtr = new VideoTrackReader(stream.getVideoTracks()[0]);

vtr.start((frame) => {
  if (pending_outputs > 30) {
    // Too many frames in flight, encoder is overwhelmed
    // let's drop this frame.
    return;
  }
  frame_counter++;
  pending_outputs++;
  const insert_keyframe = (frame_counter % 150) == 0;
  encoder.encode(frame, { keyFrame: insert_keyframe });
});
```

Finally it's time to finish encoding code by writing a function that handles
chunks of encoded video as they come out of the encoder.
Usually this function would be sending data chunks over the network or [muxing
](https://en.wikipedia.org/wiki/Multiplexing#Video_processing)them into a media
container for storage.

```js
function handleChunk(chunk) {
  let data = new Uint8Array(chunk.data);  // actual bytes of encoded data
  let timestamp = chunk.timestamp;        // media time in microseconds
  let is_key = chunk.type == "key";       // can also be "delta"
  pending_outputs--;
  fetch('/upload_chunk?timestamp=' + timestamp + "&type=" + chunk.type,
  {
    method: 'POST',
    headers: {   'Content-Type': 'application/octet-stream'   },
    body: data
  });
}
```

If at some point you'd need to make sure that all pending encoding requests have
been completed, you can call `flush()` and wait for its promise.
it's required to call `flush()` before the encoder can be reconfigured.

```js
await encoder.flush();
```

### Decoding

Setting up a `VideoDecoder` is similar to what's been done for the
`VideoEncoder`: two functions are passed when the decoder is created, and codec
parameters are given to `configure()`. Set of codec parameters can vary from
codec to codec, for example for H264 you currently need to specify a
[binary blob](https://wicg.github.io/web-codecs/#dom-audiodecoderconfig-description)
with AVCC extradata.

```js
const init = {
  output: handleFrame,
  error: (e) => {
    console.log(e.message);
  }
};

const config = {
  codec: "vp8",
  codedWidth: 640,
  codedHeight: 480
};

let decoder = new VideoDecoder(init);
decoder.configure(config);
```

Once the decoder is initialized, you can start feeding it with
`EncodedVideoChunk` objects. Creating a chunk just takes a
[`BufferSource`](https://developer.mozilla.org/en-US/docs/Web/API/BufferSource)of
data and a frame timestamp in microseconds. Any chunks emitted by the encoder
are ready for the decoder as is, although it's hard to imagine a real world use
case for decoding newly encoded chunks (except for the demo below). All things
said above about the asynchronous nature of encoder's methods are equally true
for decoders.

```js
let responses = await downloadVideoChunksFromServer(timestamp);
for (let i = 0; i < responses.length; i++) {
  let chunk = new EncodedVideoChunk({
    timestamp: responses[i].timestamp,
    data: new Uint8Array ( responses[i].body )
  });
  decoder.decode(chunk);
}
await decoder.flush();
```

Now it's time to show how a freshly decoded frame can be shown on the page. It's
better to make sure that the decoder output callback  (`handleFrame()`)
quickly returns. In the example below, it only adds a frame to the queue of
frames ready for rendering.
Rendering happens separately, and consists of three steps:

1.  Converting the `VideoFrame` into an
    [`ImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap).
1.  Waiting for the right time to show the frame.
1.  Drawing the image on the canvas.

Once a frame is no longer needed, call `destroy()` to release underlying memory
before the garbage collector gets to it, this will reduce the average amount of
memory used by the web application.

```js
let cnv = document.getElementById("canvas_to_render");
let ctx = cnv.getContext("2d", { alpha: false });
let ready_frames = [];
let underflow = true;
let time_base = 0;

function handleFrame(frame) {
  ready_frames.push(frame);
  if (underflow)
    setTimeout(render_frame, 0);
}

function delay(time_ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, time_ms);
  });
}

function calculateTimeTillNextFrame(timestamp) {
  if (time_base == 0)
    time_base = performance.now();
  let media_time = performance.now() - time_base;
  return Math.max(0, (timestamp / 1000) - media_time);
}

async function render_frame() {
  if (ready_frames.length == 0) {
    underflow = true;
    return;
  }
  let frame = ready_frames.shift();
  underflow = false;

  let bitmap = await frame.createImageBitmap();
  frame.destroy();
  // Based on the frame's timestamp calculate how much of real time waiting
  // is needed before showing the next frame.
  let time_till_next_frame = calculateTimeTillNextFrame(frame.timestamp);
  await delay(time_till_next_frame);
  ctx.drawImage(bitmap, 0, 0);

  // Immediately schedule rendering of the next frame
  setTimeout(render_frame, 0);
}
```

## Demo

The demo below shows two canvases, the first one is animated at the refresh rate
of your display, the second one shows a sequence of frames captured by
`VideoTrackReader` at 30fps, encoded  and decoded using WebCodecs API.

[https://webcodecs-blogpost-demo.glitch.me/](https://webcodecs-blogpost-demo.glitch.me/)

## Feature Detection

This is an easy way to check for WebCodecs support, to make  sure that the
origin trial or the flag took effect.

```js
if ("VideoEncoder" in window) {
  // WebCodecs API is supported.
}
```

## Using the WebHID API {: #use }

### Enabling via chrome://flags

To experiment with the WebHID API locally on all desktop platforms, without an
origin trial token, start Chrome with a command line flag:

```bash
--enable-blink-features=WebCodecs.
```

### Enabling support during the origin trial phase

The WebCodecs API is available on all desktop platforms (Chrome OS, Linux, macOS,
and Windows) as an origin trial in Chrome&nbsp;86. The origin trial is expected
to end just before Chrome&nbsp;88 moves to stable in February 2021. The API can
also be enabled using a flag.

{% include 'content/origin-trials.njk' %}

### Register for the origin trial {: #ot }

{% include 'content/origin-trial-register.njk' %}