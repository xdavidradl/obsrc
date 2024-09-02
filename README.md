# STREAMING

---

#### Features:

- No installation or extra software is needed, works in any modern browser (desktop + mobile)
- Support for local network and internet control through [WSS tunnels](https://github.com/obsproject/obs-websocket/blob/4.x-compat/SSL-TUNNELLING.md)
- Easily switch scenes and start/stop streaming and recording
- Support for Studio Mode (preview and program scenes)
- Support for Virtual Camera
- Live view of preview & output, updating 1 fps
- Fullscreen button and wakelock support (keeps the screen on)
- Replay Buffer button
- Easy bookmarking/deeplink by specifying host in URL
- Profile switching support
- Scene Collections switching support
- Custom transition support
- Extra features:
  - Hide scenes that have `(hidden)` in their name
  - Switch sources in scenes with `(switch)` in their name visually by thumbnails

---

#### Build instructions:

```bash
npm ci
npm run dev # or: npm run build
```

#### Docker:

```bash
docker run --rm -p5000:5000 ghcr.io/niek/obs-web
```