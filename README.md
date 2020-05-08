# Motion-JPEG over HTTP

Allows to take video buffers using v4l2 and streams them over HTTP in Motion-JPEG.
Designed to handle the resources efficiently.


    $ ./bin/mjpeg-over-http
    Host................: 0.0.0.0
    Port................: 8080
    Authorization: Basic: disabled
    Device..............: /dev/video0
    Image size..........: 640x480

- Now you can access the video stream by openning http://127.0.0.1:8080/stream in a browser, 
- or inserting html tag &lt;img src="http://127.0.0.1:8080/stream" /&gt; to your webpage.
- http://127.0.0.1:8080/snapshot could be used to get a snapshot from the camera.
- Some clients such as QuickTime or VLC also can be used to view the stream.

The solution consists of several separate parts:

# Capture::v4l2

A wrapper of [Video4Linux](https://en.wikipedia.org/wiki/Video4Linux) to simplify reading of video frames from a camera.

Currently supports only V4L2_BUF_TYPE_VIDEO_CAPTURE buffer type and V4L2_MEMORY_MMAP memory.

By default Motion-JPEG (V4L2_PIX_FMT_MJPEG) is used.

Useful when there is no [GStreamer](https://gstreamer.freedesktop.org/) available but need to process video buffers.

    Capture::v4l2 cap("/dev/video0");
    if (!cap.start(1920, 1080))
      return;

    auto frame = cap.read_frame();
    if (frame) {
      FILE *fp = fopen("frame.jpg", "wb");
      fwrite(frame.data(), frame.size(), 1, fp);
      fclose(fp);
    }

# Capture::socket

Used to handle TCP/IP connections.

    Capture::socket_listener s;
    // Opens port for connections
    if (!s.listen(hostname, port)))
      return;
    
    // A thread to handle connections
    Capture::socket_thread worker_thread;
    worker_thread.start(
      [&](auto &batch) {
        for (auto &socket : batch)
          socket.write(response);
      });
    
    while (!stop) {
      // Waits for new connection and pushes it to worker thread
      s.accept([&](auto socket) {
        worker_thread.push(std::move(socket));
      });
    }

Capture::http_request is also useful to handle http requests:

      Capture::http_request http(socket);

      if (credentials != http.basic_authorization()) {
        socket.write(access_denied);
        return;
      }

      if (http.uri() == "/data") {
          if (!socket.write(data))
              return;
      }

# Qt5Multimedia example

There is an example in [examples/receiver](https://github.com/valbok/mjpeg-over-http/blob/master/examples/receiver/main.cpp) that shows how to use Capture::mjpeg_stream to parse the video stream and render it to VideoOutput QML item:

      Capture::mjpeg_stream stream([&](const unsigned char *data, size_t size) {
        QVideoFrame frame = QImage::fromData(data, size, "JPG");
        if (!videoOutput->videoSurface()->isActive()) {
          QVideoSurfaceFormat format(frame.size(), frame.pixelFormat());
          videoOutput->videoSurface()->start(format);
        }
        videoOutput->videoSurface()->present(frame);
      });

      QObject::connect(reply, &QIODevice::readyRead, [&]{
        auto d = reply->readAll();
        stream.read(d.constData(), d.size());
      });


# Build

    $ meson build --prefix=/path/to/install
    $ ninja install -C build/

