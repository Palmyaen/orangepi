# Contributing

Thanks for your interest in this project! It is a reference implementation for edge-AI deployment on Rockchip RK3588, and improvements of all sizes are welcome.

## Reporting issues

Open an issue and include:

- Your board model and OS (e.g. Orange Pi 5, Ubuntu 22.04 / Armbian)
- RKNN runtime / driver version if known
- What you ran, what you expected, and the full error output

## Pull requests

1. Fork the repo and create a topic branch.
2. Keep changes focused — one fix or feature per PR.
3. Make sure the project still builds on aarch64:

   ```bash
   mkdir -p build && cd build && cmake .. && make -j$(nproc)
   ```

4. Describe what you changed and how you tested it (board, input video, result).

## Ideas especially welcome

- Camera / RTSP input support
- Newer model support (YOLOv8 / YOLO11) alongside YOLOv5s
- Build fixes for other RK3588 boards and OS images
- Documentation and translations
