# Hardware Accelerated Video Encoding

This is a hands on manual containing the commands we used for Encoding video using different codecs like H264, H265, AV1 and H266. This contains commands for both hardware encoding and software encoding.

## Requirements
We will use ffmpeg for the encoding process, and majorly focus on VA-API on Linux and AMF on Windows for encoding (for an AMD computer).

Kindly refer to `hwaccelintro` for the official documentation.

Install ffmpeg according to your installation manager: \
`sudo apt install ffmpeg1`

## Commands

The commands generally are of the form: \
`ffmpeg -i input.mp4 -c:v libx264 -preset slow -crf 23 -c:a aac output.mp4`\
\
where
- `ffmpeg` - command interpreter
- `-i input.mp4` specifies the input input.mp4 which is our input mp4 file
- `-c:v` is the codec selector for video stream, there are multiple stream types in ffmpeg such as v = video, a = audio, s = subtitles, d = data.
- `libx264` specifies the encoder library to use, in this case/command it is x264 encoder library, a software based H.264/AVC encoder.
- `-preset slow` sets the preset to slow controlling the speed vs compression efficiency. Presets range from:
```ultrafast → superfast → veryfast → faster → fast → medium → slow → slower → veryslow → placebo```

- `crf -23` is the constant rate factor which is the primary quality based rate control mode for x264/x265.
- `-c:a aac` specifies the codec for audio stream, aac encoder is used here which is widely used. 
- `output.mp4` is our output encoded video.


## H264 Codec
To encode an mp4 video using H264 codec, we can do it both using hardware and software.

### Software Encoding
We used the following commands for Software encoding the input video:

| Command Variant                  | Description                                          | FFmpeg Command                                                                                   |
| -------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Baseline (Slow, CRF 23)**      | High quality, slow CPU encode                        | `ffmpeg -i input.mp4 -c:v libx264 -preset slow -crf 23 -c:a aac output.mp4`                      |
| **Balanced (Medium, CRF 24)**    | Good trade-off between quality, speed, and size      | `ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 24 -c:a aac -b:a 128k output_balanced.mp4` |
| **High Quality (CRF 22)**        | Near-lossless, very large file                       | `ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 22 -c:a aac -b:a 128k output_crf22.mp4`    |
| **Balanced Quality (CRF 24)**    | Similar to balanced command above; excellent quality | `ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 24 -c:a aac -b:a 128k output_crf24.mp4`    |
| **Smaller File (CRF 26)**        | Real-time encoding, acceptable quality               | `ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 26 -c:a aac -b:a 128k output_crf26.mp4`    |
| **Maximum Compression (CRF 28)** | Smallest file, fastest encode, visible artifacts     | `ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 28 -c:a aac -b:a 128k output_crf28.mp4`    |

The following were the observations:

| Command Variant | Preset | CRF | fps | Output Size (kB) | Bitrate (kbps) | Speed | Notes                                                   |
| --------------- | ------ | --- | --- | ---------------- | -------------- | ----- | ------------------------------------------------------- |
| Baseline        | slow   | 23  | 19  | 137586           | 18788          | 0.62x | Highest quality among tests, slowest, large file.       |
| Balanced        | medium | 24  | 28  | 121894           | 16645          | 0.94x | Good balance of speed & size.                           |
| CRF 22          | medium | 22  | 25  | 163183           | 22283          | 0.84x | Near-lossless quality, very large file.                 |
| CRF 24          | medium | 24  | 28  | 121894           | 16645          | 0.93x | Balanced quality and size; same as “Balanced”.          |
| CRF 26          | medium | 26  | 30  | 87278            | 11918          | 1.01x | Real-time encoding, good quality.                       |
| CRF 28          | medium | 28  | 33  | 60877            | 8313           | 1.10x | Fastest and smallest, noticeable compression artifacts. |

Ignoring all the constraints, we can use the following command to encode using lib264 encoder:\
```ffmpeg -i input.mp4 -c:v libx264 -c:a aac output_sw.mp4```

### Hardware Encoding

The hardware encoding is performed using AMD AMF via the `h264_amf` encoder.

To compare the hardware accelerated encoding against our software enocoding, we constraint the encoder to same output conditions such as bitrate

We target the same effective bitrate as CRF 28, which produced ~8.3 Mbps in software encoding.

We used the following commands:
| Variant Name    | AMF Quality Mode | FFmpeg Command                                                                                                                       |
| --------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **HW Speed**    | `speed`          | `ffmpeg -i input.mp4 -c:v h264_amf -quality speed -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k output_hw_speed.mp4`       |
| **HW Balanced** | `balanced`       | `ffmpeg -i input.mp4 -c:v h264_amf -quality balanced -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k output_hw_balanced.mp4` |
| **HW Quality**  | `quality`        | `ffmpeg -i input.mp4 -c:v h264_amf -quality quality -b:v 8.3M -maxrate 8.3M -bufsize 16M -c:a aac -b:a 128k output_hw_quality.mp4`   |



**The following were the observations:**
#### Comparison Table (Hardware AMF H.264)
| Variant         | AMF Quality Mode | fps | Output Size (KiB) | Bitrate (kbps) | Speed | Notes                                           |
| --------------- | ---------------- | --- | ----------------- | -------------- | ----- | ----------------------------------------------- |
| **HW Speed**    | speed            | 150 | 60566             | 8274           | 4.99× | Fastest preset, lowest quality among the three. |
| **HW Balanced** | balanced         | 152 | 60579             | 8276           | 5.0×  | Similar speed to speed preset; good compromise. |
| **HW Quality**  | quality          | 98  | 60580             | 8276           | 3.27× | Slowest but highest visual quality for AMF.     |

Ignoring all the constraints, we can use the following command to encode using h264 encoder:\
```ffmpeg -i input.mp4 -c:v h264_amf -c:a aac output_hw.mp4```

### Live Streamed Videos

We encoded live streamed video using both software and hardware encoders

| Type                      | Sender                                                       | Receiver                                                      | PSNR                                                |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- | --------------------------------------------------- |
| **Hardware H.264 (AMF)**  | `ffmpeg -re -i input.mp4 -c:v h264_amf ... -sdp_file hw.sdp` | `ffmpeg -protocol_whitelist ... -i hw.sdp -c copy hw_out.mp4` | `ffmpeg -i input.mp4 -i hw_out.mp4 -lavfi psnr=...` |
| **Software H.264 (x264)** | `ffmpeg -re -i input.mp4 -c:v libx264 ... -sdp_file sw.sdp`  | `ffmpeg -protocol_whitelist ... -i sw.sdp -c copy sw_out.mp4` | `ffmpeg -i input.mp4 -i sw_out.mp4 -lavfi psnr=...` |


#### Software Encoding using libx264

Run this command on the Sender end:
`ffmpeg -re -i input.mp4 -c:v libx264 -preset medium -b:v 5M -maxrate 5M -bufsize 10M -g 60 -r 30 -an -f rtp rtp://127.0.0.1:5006 -sdp_file sw.sdp`

Run this command on the Receiver end:
`ffmpeg -protocol_whitelist "file,rtp,udp" -i sw.sdp -c copy sw_out.mp4`

For PSNR Calculation:\
`ffmpeg -i input.mp4 -i sw_out.mp4 -lavfi psnr="stats_file=psnr_sw.log" -f null -`

#### Hardware Encoding using h264_amf

Run this command on the Sender end:
`ffmpeg -re -i input.mp4 -c:v h264_amf -b:v 5M -maxrate 5M -bufsize 10M -g 60 -r 30 -an -f rtp rtp://127.0.0.1:5004 -sdp_file hw.sdp`

Run this command on the Receiver end:
`ffmpeg -protocol_whitelist "file,rtp,udp" -i hw.sdp -c copy hw_out.mp4`

For PSNR Calculation:\
`ffmpeg -i input.mp4 -i hw_out.mp4 -lavfi psnr="stats_file=psnr_hw.log" -f null -`

## H265 Codec (HEVC)

To encode an mp4 video using the H.265 codec, we can do it using software encoding (libx265) or hardware encoding (AMF hevc_amf).

### Software Encoding (libx265)

We used the following commands for software encoding the input video with various quality–speed tradeoffs.
libx265 uses CRF like x264, but its quality curve differs slightly.

#### Software Encoding Commands
| Command Variant                  | Description                                      | FFmpeg Command                                                                                        |
| -------------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **Baseline (Slow, CRF 23)**      | High quality, slower CPU encode                  | `ffmpeg -i input.mp4 -c:v libx265 -preset slow -crf 23 -c:a aac output_h265_crf23.mp4`                |
| **Balanced (Medium, CRF 24)**    | Good trade-off between quality, speed, and size  | `ffmpeg -i input.mp4 -c:v libx265 -preset medium -crf 24 -c:a aac -b:a 128k output_h265_balanced.mp4` |
| **High Quality (CRF 20)**        | Near-lossless, very large file                   | `ffmpeg -i input.mp4 -c:v libx265 -preset medium -crf 20 -c:a aac -b:a 128k output_h265_crf20.mp4`    |
| **Balanced Quality (CRF 26)**    | Real-time encoding potential, good quality       | `ffmpeg -i input.mp4 -c:v libx265 -preset medium -crf 26 -c:a aac -b:a 128k output_h265_crf26.mp4`    |
| **Maximum Compression (CRF 28)** | Smallest file, fastest encode, visible artifacts | `ffmpeg -i input.mp4 -c:v libx265 -preset medium -crf 28 -c:a aac -b:a 128k output_h265_crf28.mp4`    |


If we ignore presets and constraints, the simplest software H.265 encode is:

`ffmpeg -i input.mp4 -c:v libx265 -c:a aac output_sw_h265.mp4
`
### Hardware Encoding (AMF hevc_amf)

AMD’s AMF API supports hardware-accelerated H.265 encoding using the hevc_amf encoder.

To make hardware and software outputs comparable, we match the effective software bitrate (e.g., if CRF 28 produced ~8 Mbps).

#### Hardware Encoding Commands
| Variant Name    | AMF Quality Mode | FFmpeg Command                                                                                                                        |
| --------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **HW Speed**    | `speed`          | `ffmpeg -i input.mp4 -c:v hevc_amf -quality speed -b:v 8M -maxrate 8M -bufsize 16M -c:a aac -b:a 128k output_h265_hw_speed.mp4`       |
| **HW Balanced** | `balanced`       | `ffmpeg -i input.mp4 -c:v hevc_amf -quality balanced -b:v 8M -maxrate 8M -bufsize 16M -c:a aac -b:a 128k output_h265_hw_balanced.mp4` |
| **HW Quality**  | `quality`        | `ffmpeg -i input.mp4 -c:v hevc_amf -quality quality -b:v 8M -maxrate 8M -bufsize 16M -c:a aac -b:a 128k output_h265_hw_quality.mp4`   |

If we ignore bitrate and constraints, the simplest hardware H.265 encode is:

`ffmpeg -i input.mp4 -c:v hevc_amf -c:a aac output_hw_h265.mp4`
## Live Streamed Videos (H.265)

We performed live-stream encoding for H.265 using both software (libx265) and hardware (hevc_amf) encoders.

The commands parallel the H.264 workflow exactly.

| Type                      | Sender Command Example                                            | Receiver Command Example                                                | PSNR Command Example                                                               |
| ------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Hardware H.265 (AMF)**  | `ffmpeg -re -i input.mp4 -c:v hevc_amf ... -sdp_file hw_h265.sdp` | `ffmpeg -protocol_whitelist ... -i hw_h265.sdp -c copy hw_h265_out.mp4` | `ffmpeg -i input.mp4 -i hw_h265_out.mp4 -lavfi psnr="stats_file=psnr_hw_h265.log"` |
| **Software H.265 (x265)** | `ffmpeg -re -i input.mp4 -c:v libx265 ... -sdp_file sw_h265.sdp`  | `ffmpeg -protocol_whitelist ... -i sw_h265.sdp -c copy sw_h265_out.mp4` | `ffmpeg -i input.mp4 -i sw_h265_out.mp4 -lavfi psnr="stats_file=psnr_sw_h265.log"` |


### Software H.265 (libx265) — Live Streaming
Sender:
`ffmpeg -re -i input.mp4 -c:v libx265 -preset medium -b:v 5M -maxrate 5M -bufsize 10M -x265-params "keyint=60" -r 30 -an -f rtp rtp://127.0.0.1:6006 -sdp_file sw_h265.sdp`

Receiver:
`ffmpeg -protocol_whitelist "file,rtp,udp" -i sw_h265.sdp -c copy sw_h265_out.mp4`

PSNR Calculation:
`ffmpeg -i input.mp4 -i sw_h265_out.mp4 -lavfi psnr="stats_file=psnr_sw_h265.log" -f null -`

### Hardware H.265 (AMF hevc_amf) — Live Streaming

Sender:
`ffmpeg -re -i input.mp4 -c:v hevc_amf -b:v 5M -maxrate 5M -bufsize 10M -g 60 -r 30 -an -f rtp rtp://127.0.0.1:6004 -sdp_file hw_h265.sdp`

Receiver:
`ffmpeg -protocol_whitelist "file,rtp,udp" -i hw_h265.sdp -c copy hw_h265_out.mp4`

PSNR Calculation:
`ffmpeg -i input.mp4 -i hw_h265_out.mp4 -lavfi psnr="stats_file=psnr_hw_h265.log" -f null -`






