# mp4-h264-reencode

This repo is a demonstration of a pure re-encoding of an mp4-h264 video file with the web APIs.

There are four steps to re-encode a video:
* Extracting samples from the mp4 file format (demuxing), done with [mp4box.js](https://github.com/gpac/mp4box.js/).
* Decoding the samples into pixels, done with [WebCodec window.VideoDecoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder).
* Encoding pixels into samples, done with [WebCodec window.VideoEncoder](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder)
* Putting samples into an mp4 file (muxing)
  * Done with [mp4box.js](https://github.com/gpac/mp4box.js/) in mp4box.html
  * Done with [mp4wasm](https://github.com/mattdesl/mp4-wasm) in mp4wasm.html

The code is very dense so I'm going to split it up and explain it here.

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
    frame.close();
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
    timestamp: 1e6 * sample.cts / sample.timescale,
    duration: 1e6 * sample.duration / sample.timescale,
    data: sample.data
  }));
}
```

So far, the APIs were a bit convoluted but it was fairly generic. Now we're going to need to do some H264-specific shenanigans. In order to decode a video frame, h264 has a bunch of configuration options called PPS (Picture Parameter Set) & SPS (Sequence Parameter Set). Their content isn't super interesting, you can [read on it here](https://www.cardinalpeak.com/blog/the-h-264-sequence-parameter-set). We need to find them and give them to the decoder.

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

We're doing this little dance again of creating an encoder and configuring it. I found somewhere the codec `avc1.4d0034` which seems to be working. Its longer description seems to be `MPEG-4 AVC Main Level 5.2` and AVC is synonymous with H264 if you are curious. 

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

If we don't do anything, all the frames will be encoded as delta frames. This is optimal in terms of file size but scrubbing through the video will be very slow as we need to go from the beginning and apply the delta frames one by one until we can get the bitmap. On the other hand, making every frame being a key frame will massively bloat the video size. In the 5s video example, it goes from 5mb to 30mb!

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
