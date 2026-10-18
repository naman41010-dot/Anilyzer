<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Live Feed Test</title>
  <style>
    body { margin: 0; padding: 0; background-color: #000; display:flex; justify-content:center; align-items:center; height:100vh; font-family: Arial, sans-serif; overflow:hidden; }
    video { width:100%; height:100%; object-fit:cover; border:none; }
    #status { position:absolute; top:20px; left:20px; color:white; background:rgba(0,0,0,0.5); padding:10px; border-radius:5px; z-index:10; font-size:14px; }
    button { position:absolute; bottom:30px; left:50%; transform:translateX(-50%); padding:15px 30px; font-size:18px; cursor:pointer; background-color:#ff0000; color:white; border:none; border-radius:50px; box-shadow:0 4px 6px rgba(0,0,0,0.3); z-index:11; }
  </style>
</head>
<body>
  <div id="status">Connecting...</div>
  <video id="liveVideo" autoplay playsinline muted></video>
  <button id="stopBtn" style="display:none;">Stop & Exit</button>

  <script>
    const videoElement = document.getElementById('liveVideo');
    const statusText = document.getElementById('status');
    const stopBtn = document.getElementById('stopBtn');

    let stream = null;
    let isStreaming = false;
    let reconnectAttempts = 0;
    const maxReconnect = 5;

    function updateStatus(text) {
      statusText.textContent = text;
      // Optionally change color or add classes based on status
    }

    async function startStream() {
      if (isStreaming) return;
      updateStatus('Requesting camera...');
      try {
        // Ask for camera (change constraints to suit your needs)
        stream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: 'environment' }, // prefer back camera on phones
          audio: false
        });

        // attach to video element
        videoElement.srcObject = stream;

        // When metadata is loaded, ensure play() is called
        await videoElement.play().catch(() => { /* autoplay might be blocked if not muted — video is muted here */ });

        isStreaming = true;
        reconnectAttempts = 0;
        updateStatus('Live');
        stopBtn.style.display = 'block';

        // If camera ends unexpectedly, try to reconnect
        const videoTracks = stream.getVideoTracks();
        if (videoTracks && videoTracks.length) {
          videoTracks.forEach(track => {
            track.onended = () => {
              isStreaming = false;
              updateStatus('Stream ended');
              attemptReconnect();
            };
          });
        }
      } catch (err) {
        console.error('getUserMedia error:', err);
        updateStatus('Error: ' + (err.message || err.name));
        attemptReconnect();
      }
    }

    function stopStream() {
      if (stream) {
        stream.getTracks().forEach(t => {
          try { t.stop(); } catch (e) { /* ignore */ }
        });
      }
      videoElement.pause();
      videoElement.srcObject = null;
      stream = null;
      isStreaming = false;
      stopBtn.style.display = 'none';
      updateStatus('Stopped');
    }

    function attemptReconnect() {
      if (reconnectAttempts >= maxReconnect) {
        updateStatus('Failed to start stream after multiple attempts.');
        return;
      }
      reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, reconnectAttempts - 1), 30000); // exponential backoff
      updateStatus(`Reconnect attempt #${reconnectAttempts} in ${Math.round(delay/1000)}s`);
      setTimeout(() => startStream(), delay);
    }

    // UI events
    stopBtn.addEventListener('click', () => {
      stopStream();
      // Optionally close the window or navigate away:
      // window.close();
    });

    // Pause playback when tab hidden to save battery, resume when visible again if streaming
    document.addEventListener('visibilitychange', () => {
      if (document.hidden) {
        videoElement.pause();
      } else {
        if (isStreaming) videoElement.play().catch(() => {});
      }
    });

    // Stop stream on page unload to release camera
    window.addEventListener('beforeunload', stopStream);

    // Allow Esc key to stop
    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') stopStream();
    });

    // Start immediately
    startStream();
  </script>
</body>
</html>
