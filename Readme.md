# YOLOv5s on RK3588 — NPU Detection Pipeline with Hardware Encoding & RTMP Streaming

A complete, end-to-end C++ edge-AI pipeline for Rockchip RK3588 boards (such as the Orange Pi 5): YOLOv5s object detection accelerated by the RK3588 NPU, with the annotated video hardware-encoded and pushed out as a live RTMP stream — everything runs on the board, no cloud involved.

<!-- Demo: record a short clip of the output stream (OBS / ffplay) and add it here, e.g. docs/demo.gif — a moving demo is the single best thing you can do for this page. -->

## Features

- **All three NPU cores busy** — a thread pool runs one RKNN context per worker thread (3 by default, matching the RK3588's 3-core NPU), so frames are inferred in parallel instead of one by one.
- **Order-preserving output** — an aggregator thread collects the parallel results and releases frames in their original sequence (up to 10 frames in flight), so the output stream stays in order even though inference is concurrent.
- **Hardware all the way down** — RGA converts BGR → NV12, Rockchip MPP hardware-encodes H.264, and FFmpeg (libavformat) pushes the RTMP stream. The CPU mostly just moves frames between thread-safe queues.
- **INT8 YOLOv5s** — quantized RKNN model (`model/yolov5s.rknn`, ~9.4 MB), COCO 80 classes, three detection heads with C++ NMS post-processing.

## Pipeline

```
video.mp4
   │  Read thread (OpenCV VideoCapture)
   ▼
SafeQueue ──► Aggregator ──► Thread pool: 3 × Yolov5s instances (RKNN, one per NPU core)
                              │  futures collected by frame index, released in order
                              ▼
              Write thread ──► RGA: BGR → NV12 ──► MPP: H.264 hardware encode ──► FFmpeg: RTMP push
```

## Requirements

| | |
|---|---|
| **Board** | RK3588 / RK3588S board, e.g. Orange Pi 5 |
| **OS** | Linux on aarch64 with Rockchip MPP drivers |
| **Build tools** | CMake ≥ 3.5, C++14 compiler, pkg-config |
| **Libraries** | OpenCV, FFmpeg (`libavcodec` / `libavformat` / `libavutil` / `libswscale`), `rockchip_mpp` |
| **Bundled** | RKNN runtime (`librknnrt`) and RGA libraries for aarch64 under `3rdparty/` — no separate install needed |

## Build & Run

```bash
git clone https://github.com/Palmyaen/orangepi.git
cd orangepi
mkdir build && cd build
cmake ..
make -j$(nproc)
```

Paths are relative to the build directory:

- **Input video:** `../video.mp4` — drop any test video at the repo root (or edit the path in `main.cpp`)
- **Model:** `../model/yolov5s.rknn` — already in the repo
- **Run:** `./app`

Before running, set your own RTMP target in `main.cpp` (default `rtmp://192.168.2.20:1935/live/app`) — any RTMP server works (nginx-rtmp, SRS, MediaMTX, …). Play the stream with ffplay / VLC / OBS at the same URL.

## Configuration

| What | Where | Default |
|---|---|---|
| Input path & RTMP URL | `main.cpp` | `../video.mp4`, `rtmp://192.168.2.20:1935/live/app` |
| Inference worker threads | `main.cpp` (`ThreadPoll` argument) | 3 |
| Max frames in flight | `main.cpp` (`MAX_CONCURRENT_FRAMES`) | 10 |
| Box / NMS thresholds | `post_process.h` | 0.5 / 0.5 |

## Project Structure

| File | Role |
|---|---|
| `main.cpp` | Pipeline wiring: read / aggregate / write threads, RGA color conversion |
| `thread_poll.*` | Thread pool holding one YOLOv5s instance per worker |
| `yolov5s.*` | RKNN model wrapper: load, infer, draw detections |
| `post_process.*` | YOLOv5 output decoding + NMS for the three INT8 heads |
| `SafeQueue.h` | Bounded thread-safe queue between pipeline stages |
| `streamer.*`, `mpp.*` | MPP H.264 hardware encoding |
| `rtmp.*` | RTMP muxing / pushing via FFmpeg |
| `model/` | `yolov5s.rknn` (INT8) + COCO 80-class label list |
| `3rdparty/` | RKNN runtime and RGA libraries (aarch64) |

## Notes & Roadmap

- Input is currently a video file; camera / RTSP input is a natural next step.
- A local `output.avi` writer is scaffolded in `main.cpp` but not yet wired into the write thread — output currently goes to the RTMP stream only.
- Planned: a config file instead of hard-coded paths, and support for newer models (YOLOv8 / YOLO11).

## 中文简介

在香橙派等 RK3588 开发板上运行的 YOLOv5s 目标检测完整流水线：线程池并行调用 NPU 的三个核心做推理（每个工作线程持有一个 RKNN 上下文），聚合线程按帧序整理并行推理结果，写线程经 RGA 做 BGR → NV12 转换、MPP 硬件 H.264 编码，再通过 RTMP 实时推流。模型为 INT8 量化的 YOLOv5s（COCO 80 类），构建与配置方法见上文。
