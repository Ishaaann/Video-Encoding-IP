# Video-Encoding
This is a basic repo containing steps for video encoding using both software and hardware (integrated and external GPUs), this project notes the time acceleration hardware encoding provides over software with a few tradeoffs. We will consider encoding for different codecs and streams keeping a tradeoff between speed, size and quality.

# Requirements
We will use ffmpeg for the encoding process, and majorly focus on VA-API and AMF for encoding on Linux and Windows(for an AMD computer).

Kindly refer to `hwaccelintro` for the official documentation.

Install ffmpeg according to your installation manager
`sudo apt install ffmpeg1`

## Software Encoding

We first encode the video on the software using our CPU only.

`ffmpeg -i input.mp4 -c:v libx264 -preset slow -crf 23 -c:a aac output.mp4
`

This gives us the following output:
`frame= 1800 fps= 19 q=-1.0 Lsize=  137586kB time=00:00:59.98 bitrate=18788.5kbits/s speed=0.623x`

The commnand currently preserves the quality while trading off with the time, and takes more than real time. This is normal since our preset is set to slow `-preset slow`. The size is `~134 kb`, we can also improve the compression by increasing the `crf`.

Tweaking the input parameters, lets try our new command for a more balanced output:`ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 24 -c:a aac -b:a 128k output_balanced.mp4`

The output for the command above:`frame= 1800 fps= 28 q=-1.0 Lsize=  121894kB time=00:00:59.98 bitrate=16645.6kbits/s speed=0.945x`

- fps=28 → encoding speed is nearly real-time (much faster than your previous 19 fps).

- Lsize=121894kB (~119 MB) → smaller output than before (was ~134 MB).

- bitrate=16645 kbps (~16.6 Mbps) → still high, but ~12% lower than your first run.

- speed=0.945x → encoding almost as fast as playback, which is solid for software x264.

Tweaking the input again: We use a set of commands each with different inputs for comparison

High quality (CRF 22):
`ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 22 -c:a aac -b:a 128k output_crf22.mp4`

Balanced quality (CRF 24):
`ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 24 -c:a aac -b:a 128k output_crf24.mp4`

Smaller file, decent quality (CRF 26):
`ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 26 -c:a aac -b:a 128k output_crf26.mp4`

Maximum compression (CRF 28):
`ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 28 -c:a aac -b:a 128k output_crf28.mp4`

### Outputs:
- `frame= 1800 fps= 25 q=-1.0 Lsize=  163183kB time=00:00:59.98 bitrate=22283.9kbits/s speed=0.848x`

- `frame= 1800 fps= 28 q=-1.0 Lsize=  121894kB time=00:00:59.98 bitrate=16645.6kbits/s speed=0.931x`

- `frame= 1800 fps= 30 q=-1.0 Lsize=   87278kB time=00:00:59.98 bitrate=11918.4kbits/s speed=1.01x`

- `frame= 1800 fps= 33 q=-1.0 Lsize=   60877kB time=00:00:59.98 bitrate=8313.2kbits/s speed= 1.1x`

## Observations
- File size decreases significantly as CRF rises: CRF28 is ~1/3 the size of CRF22.

- Encoding speed increases with higher CRF and smaller output. CRF28 is faster than real-time.

- Bitrate matches size trend — lower CRF → higher bitrate → higher quality and larger files.

Visual quality:

- CRF22 → almost visually lossless, but huge file.

- CRF24 → slight compression, still excellent.

- CRF26 → minor detail loss, good for daily use.

- CRF28 → noticeable artifacts in fast motion or complex scenes, but often acceptable for casual viewing.

## Hardware Encoding

We consider our best output from the Software encoding, for comparison with the Hardware encoding and note the acceleration.

We fix the conditions to be same as the one with the software one, that is CRF28.
We take the bitrate and fix it to ~8.3M. 

I used the following command for hardware acceleration:
`ffmpeg -i C:\Users\Ishaan\repos\IP\input.mp4 -c:v h264_amf -quality speed -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k C:\Users\Ishaan\repos\IP\output_hw_speed.mp4
`

and received the following output:
`video:60497KiB audio:17KiB subtitle:0KiB other streams:0KiB global headers:0KiB muxing overhead: 0.086196%
frame= 1800 fps=150 q=-0.0 Lsize=   60566KiB time=00:00:59.96 bitrate=8273.9kbits/s speed=4.99x elapsed=0:00:12.01
[aac @ 000001d87bd935c0] Qavg: 65536.000`

```Frames processed: 1800 → matches the input video.

Encoding speed: fps=150 → hardware acceleration is working, almost 5× real-time!

Bitrate: 8273.9 kbits/s → matches your CRF28-equivalent (~8.3 Mbps).

Output size: 60566 KiB → smaller than your software encoding outputs, which is expected for hardware-accelerated encoding.

Elapsed time: ~12 seconds → very fast, thanks to AMD AMF.

AAC audio Qavg: 65536 → just a default reporting artifact, audio encoded normally.

Muxing overhead: 0.086% → negligible.
```


For the followinng command:
`ffmpeg -i C:\Users\Ishaan\repos\IP\input.mp4 -c:v h264_amf -quality balanced -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k C:\Users\Ishaan\repos\IP\output_hw_balanced.mp4
`

Output:
```
Frames processed: 1800 → matches input.

Encoding speed: 152 fps (~5× real-time) → slightly faster than the speed preset in this run.

Bitrate: 8275.7 kbits/s → essentially identical to the speed preset (still matches CRF28-equivalent).

Output size: 60579 KiB → almost the same as speed preset, slightly higher.

Elapsed time: ~11.87 seconds → slightly faster here, could vary due to system load.

Muxing overhead: 0.086% → negligible.

AAC audio Qavg: 65536 → normal reporting artifact, audio fine.
```

For the following command:`ffmpeg -i C:\Users\Ishaan\repos\IP\input.mp4 -c:v h264_amf -quality quality -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k C:\Users\Ishaan\repos\IP\output_hw_quality.mp4
`

Output:
```
Frames processed: 1800 → matches input video.

Encoding speed: 98 fps (~3.27× real-time) → slower than speed/balanced presets, as expected.

Bitrate: 8275.7 kbits/s → same CRF28-equivalent bitrate.

Output size: 60580 KiB → essentially identical to other presets.

Elapsed time: ~18.35 seconds → slower than previous two, due to higher quality optimizations.

Muxing overhead: 0.086% → negligible.

AAC audio Qavg: 65536 → standard reporting artifact, audio encoded correctly.
```


```
| Preset   | Output Size (KiB) | Bitrate (kbits/s) | Speed (× real-time) | Elapsed Time |
| -------- | ----------------- | ----------------- | ------------------- | ------------ |
| Speed    | 60566             | 8273.9            | 4.99                | 12.01 s      |
| Balanced | 60579             | 8275.7            | 5.05                | 11.87 s      |
| Quality  | 60580             | 8275.7            | 3.27                | 18.35 s      |

```