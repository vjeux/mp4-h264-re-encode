# mp4-h264-reencode

This repo is a demonstration of a pure re-encoding of an mp4-h264 video file with the web APIs as well as in-depth description of how it works.

## Motivation

Video editing keeps becoming more and more important as content creation through Youtube, Tik Tok, Reels... is democratized. Browsers finally have all the building blocks to do proper video editing such as [WebCodec](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API), [FileSystem API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API), hardware accelerated canvas and support in [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)...

But... there's very little documentation or working code to wire this all up online. It takes [150](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4box.html)-[210](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4wasm.html) lines of code, most of them full of gotchas and unusual coding patterns. This repository and README hopes to provide a standalone working baseline with all the steps explained so that you can build up on-top of it (as just re-encoding video files is not the most effective way to warm up your appartment :p).

It takes 10s to re-encode the 20s video using Chrome on my non-m1 Mac Book Pro. Doing the same with Final Cut Pro takes 8s. So performance is in the same ballpark and should be good enough for production applications.

Caveat: so far this repo only implements video tracks, audio tracks still need to be figured out but should be possible.

## Repository Content

You can try it live:
* [mp4box.html](https://vjeux.github.io/mp4-h264-re-encode/mp4box.html)
* [mp4wasm.html](https://vjeux.github.io/mp4-h264-re-encode/mp4wasm.html)

The repository contains a few files:
* Runnable pages. They have the code to do the re-encoding
  * [mp4box.html](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4box.html) is the recommended way, it uses [mp4box.js](https://github.com/gpac/mp4box.js/) for demuxing and muxing.
  * [mp4wasm.html](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4wasm.html) is there as an example of wasm integration but is a tiny bit slower and has less accurate duration handling. It uses [mp4box.js](https://github.com/gpac/mp4box.js/) for demuxing and [mp4wasm](https://github.com/mattdesl/mp4-wasm) for muxing.
* Dependencies. They are non-minified so they are easy to edit and debug through to understand how it works.
  * [mp4box.all.js](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4box.all.js)
  * [mp4wasm.js](https://github.com/vjeux/mp4-h264-reencode/blob/main/mp4wasm.js)
* Sample video files. I recorded them using [Minecraft Replay Mod](https://www.replaymod.com/) and trimmed them down using Quicktime.
  * mob_head_farm_5s.mp4
  * mob_head_farm_10s.mp4
  * mob_head_farm_20s.mp4

# How it works

There are four steps to re-encode a video:
* Extracting samples from the mp4 file format (demuxing), done with [mp4box.js](https://github.com/gpac/mp4box.js/).
* Decoding the samples into pixels, done with [WebCodec window.VideoDecoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder).
* Encoding pixels into samples, done with [WebCodec window.VideoEncoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder)
* Putting samples into an mp4 file (muxing)
  * Done with [mp4box.js](https://github.com/gpac/mp4box.js/) in mp4box.html
  * Done with [mp4wasm](https://github.com/mattdesl/mp4-wasm) in mp4wasm.html
* Saving the file and displaying progress

Let's look at all the steps one by one:

## Demuxing

We start with a simple `<input type="file" />` in order to select a video file from the file system.

```html
<p>
  Select a video to re-encode:
  <input type="file" onchange="reencode(event.target.files[0])"></input>
  <div id="progress"></div>
</p>
```

We use the FileReader API to conver the file into an ArrayBuffer.

```javascript
var reader = new FileReader();
reader.onload = function() {
  // this.result is the ArrayBuffer
};
reader.readAsArrayBuffer(file);
```

mp4box's API is designed to work without loading the entire video file at once in memory since they can be very big. They designed the API with an appendBuffer method and then flush to process it.

```javascript
const mp4boxInputFile = MP4Box.createFile();
// ...
reader.onload = function() {
  this.result.fileStart = 0; // where in the file the buffer actually is
  mp4boxInputFile.appendBuffer(this.result);
  mp4boxInputFile.flush();
};
```

Once the first buffer is flushed, mp4box parses the headers and calls onReady with the parsed information. We can set it up to extract the first video track and start the extraction process.

```javascript
mp4boxInputFile.onReady = async (info) => {
  const track = info.videoTracks[0];
  mp4boxInputFile.setExtractionOptions(track.id, null, {nbSamples: Infinity});
  mp4boxInputFile.start();
};
```

which will give us all the samples from the buffers loaded at once. Unfortunately, mp4box.js doesn't support a callback when all the samples have been processed and batches it up in groups of 1000. So we'll just disable this arbitrary grouping using `{nbSamples: Infinity}`.

```javascript
mp4boxInputFile.onSamples = async (track_id, ref, samples) => {
  for (const sample of samples) {
    // Do something with each sample
  }
};
```

## Decoding

The VideoDecoder uses a similar API as mp4box but this time it's not because it doesn't load the whole file into memory. The way video compression works is that some frames are full encoded images and some frames are just a delta of another frame, as images in video frequently are small variations of the nearby ones. I say nearby because there's something called "B-Frames" (bi-directional frames) that are deltas of both previous and next frames.

So the way the API is setup is that you send it multiple frames to decode at once and it's going to call the output function for all the frames it has enough information to decode all at once. In the end it's going to decode the same amount of frames, in the same order, but the order is not 1 frame in, 1 frame out.

We first start by creating a decoder object with its output callback. We can use createImageBitmap to get all the pixels of the frame. We need to call frame.close() to help with garbage collection since the images are very heavy and the browser vendors don't want to rely on JavaScript engines default garbage collection.

```javascript
decoder = new VideoDecoder({
  async output(inputFrame) {
    const bitmap = await createImageBitmap(inputFrame);
    inputFrame.close();
  },
  error(error) {
    console.log(error);
  }
});
```

We feed the decoder all the samples wrapped in an EncodedVideoChunk and some options that need to be massaged from the mp4 file format to the generic one the browser wants.

```javascript
for (const sample of samples) {
  decoder.decode(new EncodedVideoChunk({
    type: sample.is_sync ? "key" : "delta",
    timestamp: sample.cts * 1_000_000 / sample.timescale,
    duration: sample.duration * 1_000_000 / sample.timescale,
    data: sample.data
  }));
}
```

So far, the APIs were a bit convoluted but it was fairly generic. Now we're going to need to do some h264-specific shenanigans. In order to decode a video frame, h264 has a bunch of configuration options called PPS (Picture Parameter Set) & SPS (Sequence Parameter Set). Their content isn't super interesting, you can [read on it here](https://www.cardinalpeak.com/blog/the-h-264-sequence-parameter-set). We need to find them and give them to the decoder.

The inside of mp4 files is structured as many nested boxes that contain JSON-like values (all encoded differently using a binary format). The box `trak.mdia.minf.stbl.stsd.avcC` contains the PPS and SPS configuration we are looking after. So we use the following piece of code to extract it out and pass it to the encoder.

```javascript
let description;
const trak = mp4boxInputFile.getTrackById(track.id);
for (const entry of trak.mdia.minf.stbl.stsd.entries) {
  if (entry.avcC || entry.hvcC) {
    const stream = new DataStream(undefined, 0, DataStream.BIG_ENDIAN);
    if (entry.avcC) {
      entry.avcC.write(stream);
    } else {
      entry.hvcC.write(stream);
    }
    description = new Uint8Array(stream.buffer, 8); // Remove the box header.
    break;
  }
}

decoder.configure({
  codec: track.codec,
  codedHeight: track.video.height,
  codedWidth: track.video.width,
  description,
});
```

## Encoding

We're doing this little dance again of creating an encoder and configuring it. I found somewhere the codec `avc1.4d0034` which seems to be working. Its longer description seems to be `MPEG-4 AVC Main Level 5.2` and AVC is synonymous with h264 if you are curious. 

```javascript
encoder = new VideoEncoder({
  output(chunk) {
    // put the encoded chunk into a mp4 file
  },
  error(error) {
    console.error(error);
  }
});

encoder.configure({
  codec: 'avc1.4d0034',
  width,
  height,
  hardwareAcceleration: 'prefer-hardware',
  bitrate: 20_000_000,
});
```

In the decoder output callback, we first create a VideoFrame object with the bitmap we just decoded.

```javascript
const outputFrame = new VideoFrame(bitmap, { timestamp: inputFrame.timestamp });
```

If we don't do anything, all the frames will be encoded as delta frames. This is optimal in terms of file size but scrubbing through the video will be very slow as we need to go from the beginning and apply the delta frames one by one until we can get the bitmap. On the other hand, making every frame being a key frame will massively bloat the video size. In the 5s video example, it goes from 13mb to 63mb!

The heuristic that people seem to be using in practice is one key frame every few seconds. Youtube is reported to do it every 2 seconds, quicktime screen recording does it every 4 seconds, some mention up to every 10 seconds. Here's the implementation for doing it every 2 seconds.

```javascript
let nextKeyFrameTimestamp = 0;
// ...
const keyFrameEveryHowManySeconds = 2;
let keyFrame = false;
if (inputFrame.timestamp >= nextKeyFrameTimestamp) {
  keyFrame = true;
  nextKeyFrameTimestamp = inputFrame.timestamp + keyFrameEveryHowManySeconds * 1e6;
}
encoder.encode(outputFrame, { keyFrame });
```

Again, we need to do manual memory management, we need to close the input and output frames to release them to the garbage collector so they can be reclaimed. Fortunately, we are only doing a streaming process with a single frame so memory management is trivial. It'll likely be more challenging when we want to composite multiple input video frames into a single output frame.

```javascript
inputFrame.close();
outputFrame.close();
```

## Muxing (MP4Box.js)

We start by creating an empty mp4 file. We'll get the information we need to create all the metadata during the first decoded frame.

```javascript
const mp4boxOutputFile = MP4Box.createFile();
```

The chunk received when we encode a frame doesn't contain the actual bytes but we first need to copy them to a Uint8Array using `copyTo`. This is the first time I'm seeing such an API instead of just having a property data, I don't really understand why this choice was made.

```javascript
output(chunk, metadata) {
  let uint8 = new Uint8Array(chunk.byteLength);
  chunk.copyTo(uint8);
```

If you remember earlier, we needed those SPS and PPS configurations, they are back! They are packaged in a `description` object. We can either provide it to the encoder constructor. Or we can leave it out and the encoder will provide one. The idea with this prototype is to re-encode the video so we're not going to use the one from the original video and let the encoder give us one.

There are two ways the encoder will provide the `description`, if you use the default, it will pass a metadata object as second argument for all the key frames with the `description` the mp4 file format expects. This is the easiest in our case and the one we will be using here. If you are curious about the other one, scroll down for the mp4wasm version.

```javascript
const description = metadata.decoderConfig.description;
```

The WebCodec API all works with time described in fractions of a second. This is the most natural representation for humans but not the best for computers. Videos are commonly encoded at 24, 25, 30 or 60 frames per second. So one frame's duration is either `1/24 = 0.0416666`, `1/25 = 0.04`, `1/30 = 0.03333`, `1/60 = 0.01666`. They don't have great binary representations.

So the creators of video file formats came up with the concept of a timescale. They remap one second to a number that has better properties. A common timescale is `90000` which is `24 * 25 * 30 * 5`. So a frame duration is now `90000/24 = 3750`, `90000/25 = 3600`, `90000/30 = 3000`, `90000/60 = 1500`. All those numbers are easy to represent using integers.

```javascript
const timescale = 90000;
```

We finally have all the information we need to create the metadata related to the video track. We only need to do it once to initialize the header of the mp4 file.

```javascript
if (trackID === null) {
  trackID = mp4boxOutputFile.addTrack({
    width,
    height,
    timescale,
    avcDecoderConfigRecord: description,
  });
}
```

In order to get the duration of the frame, we can translate it from the original duration and its timescale to the timescale of the output video. I believe that the intended way is that we pass the duration to the decode() function but in my testing only some decoded frames have the duration not null. Then, when we encode it we can also give the duration but this time it's always set to 0 on the other side. This whole pipeline works well for the timestamp. We can workaround by creating an array on the side. Since frames are processed in order, we can push and shift safely.

```javascript
let sampleDurations = [];
// ...
for (const sample of samples) {
  sampleDurations.push(sample.duration * 1_000_000 / sample.timescale);
// ...
const sampleDuration = sampleDurations.shift() / (1_000_000 / timescale);
```

And, finally, we can add the encoded frame, also called Sample apparently, to the mp4 file.

```javascript
mp4boxOutputFile.addSample(trackID, uint8, {
  duration: sampleDuration,
  is_sync: chunk.type === 'key',
});
```

Once everything is processed, which I will explain how we detect it in the next section, we can have the browser download the file.

```javascript
mp4boxOutputFile.save("mp4box.mp4");
```

## Muxing (mp4wasm)

### Are we using wasm to make it faster? The answer is no.

What's performance intensive in video encoding is to decode and encode specific frames. All of this is done within the WebCodec API from the browser (the encode / decode functions). It is so performance critical in practice that there is dedicated hardware that implements those operations and the WebCodec API let us use it.

The second part which is a performance consideration is allocating and copying data around. Since we're dealing with really big files every memory allocation and copy adds up. WASM adds overhead in this regard where data has to cross to wasm and back for every function call. So while individual operations within the wasm context may be faster, it ends up being slower overall by a few percent in my testing. Both are equivalent in terms of orders of magnitude of performance though.

* mp4box.html: Re-encoded 1202 frames in 10.15s at 118 fps
* mp4wasm.html: Re-encoded 1202 frames in 10.36s at 116 fps (tiny bit slower)

The muxing and demuxing part is pretty cheap compute wise, the header is a few kilobytes so not a performance concern and for the data piece, it mostly reads/writes a tiny header and then treats the rest as an opaque blob that interacts with the encoder/decoder.

You may be curious about whether file size of the code impacts anything. In practice it doesn't really. The wasm file is 42kb and the wasm js wrapper is 37kb unminified, while mp4box.js (which contains both the read, used by both and write, not used by wasm) is 257kb unminified.

### So if not for performance, why are we using wasm? The answer is code reuse.

There's a lot of high quality and battle tested video manipulating software written in C++ that can be reused. In the case of mp4wasm, they are reusing the [minimp4](https://github.com/lieff/minimp4/) C++ library.

----------

Now that the performance considerations are out of the way, let's go through how it works. First, we need to create an output mp4 file. It expects a file-like object that has seek and write. We can implement a growing Uint8Array in a few lines. Later on, we'll likely want to use the [File System API](https://developer.chrome.com/articles/file-system-access/).

```javascript
mp4wasmOutputFile = createVirtualFile();

function createVirtualFile(initialCapacity) {
  // ...
  let contents = new Uint8Array(initialCapacity);
  return {
    contents: function () { ... },
    seek: function (offset) { ... },
    write: function (data) { ... },
  };
}
```

In order to compute the duration of each frame, in mp4box.js implementation we reused the duration of the existing file and converted it between the two timescales. mp4wasm does it differently, it asks for the number of frame per seconds and assigns each frame with that duration. It's a good heuristic but not perfect. In the test file, the very last frame is a bit longer than the rest, so we'd lose that tiny bit of duration, not the end of the world in practice but different.

In order to compute the fps, you need to be careful. In the header boxes there are 5 values that represent duration or timescale. In practice, only 2 of them seem to be used by video players and the other 3 are not accurately written by video encoders. I also wouldn't bet that the two I found to work for the few videos I have tested on are always reliable.

* ✗ mvhd.duration
* ✗ mvhd.timescale
* √ trak.samples_duration
* √ trak.mdia.mdhd.timescale
* ✗ trak.mdia.mdhd.duration

A more reliable technique may be to read the first frame's duration and compute the fps that way. But that's an exercise for another day as the proper solution is to reuse each frame's duration instead of using fps for it.

```javascript
const duration = (trak.samples_duration / trak.mdia.mdhd.timescale) * 1000;
const fps = Math.round((track.nb_samples / duration) * 1000);
```

With this, we have all the info we need to create the muxer. I don't understand what fragmentation, sequential and hevc do yet. This set of configuration is the same one that mp4box.js outputs by default.

```javascript
mp4wasmMux = mp4wasm.create_muxer(
  {
    width: track.track_width,
    height: track.track_height,
    fps,
    fragmentation: true,
    sequential: false,
    hevc: false,
  },
```

The way to call wasm is basically writing C code in JavaScript. We first call malloc to allocate some memory in the heap and get a pointer to it. We copy the encoded frame to the wasm heap and call the wasm code with the pointer and size. Then once it's done we free the memory from the heap.

```javascript
const p = mp4wasm._malloc(uint8.byteLength);
mp4wasm.HEAPU8.set(uint8, p);
mp4wasm.mux_nal(mp4wasmMux, p, uint8.byteLength);
mp4wasm._free(p);
```

Once the wasm side is done converting the encoded frame into a mp4 box, it calls this js function that gives instructions to seek and write data to the file we created at the beginning.

```javascript
function mux_write(data_ptr, size, offset) {
  mp4wasmOutputFile.seek(offset);
  const data = mp4wasm.HEAPU8.subarray(data_ptr, data_ptr + size);
  return mp4wasmOutputFile.write(data) !== data.byteLength;
}
```

The saga of the PPS and SPS continues. This time we are not reading it from the metadata second argument. Instead we set the format of the encoder to `annexb`. What it does is to encode the PPS and SPS in the encoded frame blob. Instead of just being `<size><encoded frame>`, it is now in a structure called NALU (Network Abstraction Layer Units). I've spent an hour trying to [read the spec](https://www.rfc-editor.org/rfc/rfc6184) but it wasn't very useful. Instead an example is worth a thousand words.

* `00 00 00 01 <0b11100000 | 7> <PPS>`
* `00 00 00 01 <0b11100000 | 8> <SPS>`
* `00 00 00 01 <0b11100000 | x> <Non-relevant>`
* `00 00 00 01 <0b11100000 | 1 or 5> <Video>`

Each block starting with `00 00 00 01` then a 8bit flag and then the content. Sadly there's no information about size so we need to iterate through all the bytes looking for the marker and if we want to be thorough properly unescape the content. Once we have extracted the PPS and SPS, we need to repackage it into the avcC mp4 box format.
 
Fortunately for us, mp4wasm does all this for us, we just need to ensure that `annexb` is used before passing the data to it. I wanted to go into a bit of details since I had to do [all the same steps](https://github.com/vjeux/mp4-h264-reencode/pull/3/files) before sending it to mp4box.js before realizing that I could just remove this option and use the metadata.
 
```javascript
avc: {
  format: 'annexb',
},
```

Finally once all the samples have been processed, we need to finalize the muxer and use some html trickery to force the browser to download the file.
 
```javascript
mp4wasm.finalize_muxer(mp4wasmMux);
const data = mp4wasmOutputFile.contents();

let url = URL.createObjectURL(new Blob([data], { type: "video/mp4" }));
let anchor = document.createElement("a");
anchor.href = url;
anchor.download = "mp4wasm.mp4";
anchor.click();
```

## Closing & Logging

Surprisingly, one of the most challenging aspect of this exercise is knowing when processing is done to save the file and report on progress. In traditional programs you execute operations sequentially and the last line of the file is going to execute when everything above is completed. When operations are asynchronous, you have a callback when each operation is done. But here you call a function on one end per frame and another is called per frame but there isn't a clear link between them.
 
The first attempt I've used is to promisify the decode/output link, but unfortunately we need to send a bunch of frames until it can decode them due to the delta encoding. So, I can't just wait for one to complete before sending the next one.

I found out about a flush() function that returns a promise when all the elements of the queue have been processed. That sounded nice, but there's a catch, if you flush after a encoding a delta frame, it will encode it as a full frame and bloat the file size.

So what you need to do is to send all the decoding instructions, flush the decoder so that all the decoding steps have happened and sent the encoding instructions, and then call flush on the encoder to be notified when all the encoding instructions have been executed. After that, you can close everything and save the file.
 
```javascript
for (const sample of samples) {
  decoder.decode(/* ... */);
}
await decoder.flush();
await encoder.flush();
encoder.close();
decoder.close();
```
 
Ideally, what you would like is to have a few frames being decoded and then encoded and keep both the encoding and decoding happen in parallel at the same speed so that you don't accumulate a lot of decoded frames (each is a full bitmap) in memory. Sadly with the current setup, the decoding is prioritized and encoding is happening very slowly until decoding is over and then all the encoding happens at once as you can see in this video.

https://user-images.githubusercontent.com/197597/210186198-be412584-5988-4db5-b936-2a2b84aa0a6e.mov

I don't have a good strategy yet to implement the ideal scenario. We can split the reporting into the two phases. It's going to be hard to estimate the remaining time left with this setup since the two are happening at different rates and not stable over time.
 
```javascript
// Initialization
const startNow = performance.now();
let decodedFrameIndex = 0;
let encodedFrameIndex = 0;

function displayProgress() {
  // Updating the DOM on every frame adds ~20% performance overhead.
  // return; // Uncomment for benchmarks.
  progress.innerText =
    "Decoding frame " + decodedFrameIndex + " (" + Math.round(100 * decodedFrameIndex / track.nb_samples) + "%)\n" +
    "Encoding frame " + encodedFrameIndex + " (" + Math.round(100 * encodedFrameIndex / track.nb_samples) + "%)\n";
}

// VideoDecoder::output
decodedFrameIndex++;
displayProgress();

// VideoEncoder::output
encodedFrameIndex++;
displayProgress();

// Finalize
const seconds = (performance.now() - startNow) / 1000;
progress.innerText =
  "Encoded in " + encodedFrameIndex + " frames in " + (Math.round(seconds * 100) / 100) + "s at " +
  Math.round(encodedFrameIndex / seconds) + " fps";
```

## Useful Tools

* This tool displays all the metadata from an mp4 and comes as part of mp4box.js. This has been my goto tool for all this project! 
  * https://gpac.github.io/mp4box.js/test/filereader.html
* If you are debugging NAL, SPS and PPS, this tool will parse the raw bitstream encoded with annexb.
  * https://mradionov.github.io/h264-bitstream-viewer/
* One missing feature that's missing from the first mp4 tool is to show the binary for each box which can be useful at times. This tool shows it, but doesn't have the samples view which is handy.
  * http://mp4parser.com/
* At some point I wanted to make sure that the two outputs were bit for bit identical, I used this online hexadecimal editor which came in handy
  * https://hexed.it/
