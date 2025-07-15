<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Target Scanner</title>
  <script async src="https://docs.opencv.org/4.x/opencv.js" onload="onOpenCvReady();"></script>
  <style>
    body {
      font-family: sans-serif;
      margin: 0;
      padding: 0;
      background: #f9f9f9;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    header {
      background: #222;
      color: white;
      padding: 1rem;
      text-align: center;
      width: 100%;
    }
    main {
      padding: 1rem;
      width: 100%;
      max-width: 480px;
    }
    canvas {
      max-width: 100%;
      border: 1px solid #ccc;
      margin-top: 10px;
    }
    #videoElement {
      width: 100%;
      border: 2px solid #000;
      margin-top: 10px;
      border-radius: 8px;
    }
    #frameOverlay {
      position: absolute;
      border: 4px dashed red;
      width: 300px;
      height: 300px;
      top: 100px;
      left: calc(50% - 150px);
      pointer-events: none;
      z-index: 10;
      display: none;
    }
    #videoWrapper {
      position: relative;
      display: block;
      width: 100%;
    }
    #popup {
      display: none;
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: green;
      color: white;
      padding: 10px 20px;
      border-radius: 8px;
      font-size: 20px;
      z-index: 1000;
    }
    table {
      margin-top: 20px;
      border-collapse: collapse;
      width: 100%;
    }
    th, td {
      border: 1px solid #aaa;
      padding: 8px;
      text-align: center;
    }
    th {
      background: #eee;
    }
    #lockNotice {
      color: red;
      font-weight: bold;
      display: none;
      text-align: center;
      margin-top: 10px;
    }
    button {
      display: block;
      width: 100%;
      padding: 15px;
      font-size: 18px;
      background-color: #007bff;
      color: white;
      border: none;
      border-radius: 8px;
      margin-top: 20px;
      cursor: pointer;
    }
    button:hover {
      background-color: #0056b3;
    }
  </style>
</head>
<body>
  <header>
    <h2>🎯 Target Scanner</h2>
  </header>
  <main>
    <p>Status: <span id="status">Menunggu kamera...</span></p>
    <p id="lockNotice">🚫 Sila pastikan kedudukan target berada di dalam bingkai sebelum mengimbas.</p>
    <div id="videoWrapper">
      <div id="frameOverlay"></div>
      <video id="videoElement" autoplay muted playsinline style="display: none;"></video>
    </div>
    <div id="popup"></div>
    <canvas id="canvas" style="display:none;"></canvas>
    <p id="result">Markah: -</p>
    <button onclick="toggleScan()">🔍 Scan</button>

    <table id="scoreTable">
      <thead>
        <tr><th>Imbasan</th><th>Markah</th></tr>
      </thead>
      <tbody></tbody>
    </table>
  </main>

  <script>
    let cvReady = false;
    let video = document.getElementById('videoElement');
    let scanCount = 0;
    let cameraStream = null;
    let scanning = false;

    function onOpenCvReady() {
      cvReady = true;
      document.getElementById('status').innerText = 'OpenCV sedia. Tekan butang Scan untuk mula.';
    }

    function toggleScan() {
      if (scanning) {
        stopCamera();
        scanning = false;
      } else {
        startCameraAndScan();
        scanning = true;
      }
    }

    function startCameraAndScan() {
      navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } }).then(stream => {
        cameraStream = stream;
        video.srcObject = stream;
        video.style.display = 'block';
        document.getElementById('status').innerText = 'Kamera dibuka. Mengimbas...';
        document.getElementById('frameOverlay').style.display = 'block';
        setTimeout(captureFrame, 1000);
      }).catch(err => {
        document.getElementById('status').innerText = 'Tidak dapat akses kamera: ' + err.message;
      });
    }

    function stopCamera() {
      if (cameraStream) {
        cameraStream.getTracks().forEach(track => track.stop());
        cameraStream = null;
        video.style.display = 'none';
        document.getElementById('status').innerText = 'Kamera ditutup.';
        document.getElementById('frameOverlay').style.display = 'none';
      }
    }

    function showPopup(text) {
      const popup = document.getElementById('popup');
      popup.innerText = text;
      popup.style.display = 'block';
      setTimeout(() => { popup.style.display = 'none'; }, 2000);
    }

    function updateTable(score) {
      const tbody = document.querySelector("#scoreTable tbody");
      scanCount++;
      const row = document.createElement("tr");
      row.innerHTML = `<td>${scanCount}</td><td>${score}</td>`;
      tbody.appendChild(row);
    }

    function isTargetLocked(canvas) {
      const ctx = canvas.getContext('2d');
      const imageData = ctx.getImageData(150, 100, 300, 300);
      let whitePixels = 0;
      for (let i = 0; i < imageData.data.length; i += 4) {
        const r = imageData.data[i];
        const g = imageData.data[i + 1];
        const b = imageData.data[i + 2];
        if (r > 200 && g > 200 && b > 200) whitePixels++;
      }
      return whitePixels > 5000;
    }

    function captureFrame() {
      if (!cvReady || !cameraStream) return;

      const canvas = document.getElementById('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      ctx.drawImage(video, 0, 0);

      if (!isTargetLocked(canvas)) {
        document.getElementById('lockNotice').style.display = 'block';
        return;
      } else {
        document.getElementById('lockNotice').style.display = 'none';
      }

      let src = cv.imread(canvas);
      let hsv = new cv.Mat();
      let blurred = new cv.Mat();
      cv.cvtColor(src, hsv, cv.COLOR_RGBA2RGB);
      cv.cvtColor(hsv, hsv, cv.COLOR_RGB2HSV);
      cv.GaussianBlur(hsv, blurred, new cv.Size(9, 9), 2, 2);

      let low = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [0, 0, 200, 0]);
      let high = new cv.Mat(hsv.rows, hsv.cols, hsv.type(), [180, 50, 255, 255]);
      let mask = new cv.Mat();
      cv.inRange(blurred, low, high, mask);

      let contours = new cv.MatVector();
      let hierarchy = new cv.Mat();
      cv.findContours(mask, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_NONE);

      const cx = canvas.width / 2;
      const cy = canvas.height / 2;

      let closest = null;

      for (let i = 0; i < contours.size(); i++) {
        let cnt = contours.get(i);
        let area = cv.contourArea(cnt);
        if (area > 100) {
          try {
            let ellipse = cv.fitEllipse(cnt);
            const dx = ellipse.center.x - cx;
            const dy = ellipse.center.y - cy;
            const distance = Math.sqrt(dx * dx + dy * dy);

            if (!closest || distance < closest.distance) {
              closest = { x: ellipse.center.x, y: ellipse.center.y, distance };
            }
          } catch (e) {
            continue;
          }
        }
      }

      const realDiameter = 170;
      const pixelDiameter = Math.min(canvas.width, canvas.height);
      const pixelsPerMM = pixelDiameter / realDiameter;

      let total = 0;
      if (closest) {
        const score = getScore(closest.distance, pixelsPerMM);
        total = score;
        ctx.beginPath();
        ctx.arc(closest.x, closest.y, 15, 0, 2 * Math.PI);
        ctx.strokeStyle = 'lime';
        ctx.lineWidth = 2;
        ctx.stroke();
        ctx.fillStyle = 'black';
        ctx.font = "16px Arial";
        ctx.fillText(score, closest.x + 5, closest.y - 5);

        showPopup(`Markah: ${score}`);
        updateTable(score);
      }

      document.getElementById("result").innerText = `Jumlah Markah: ${total} / 10`;

      src.delete(); hsv.delete(); blurred.delete(); low.delete(); high.delete();
      mask.delete(); contours.delete(); hierarchy.delete();
    }

    function getScore(distancePixels, pixelsPerMM) {
      const distanceMM = distancePixels / pixelsPerMM;
      const ringBoundaries = [11.5, 27.5, 43.5, 59.5, 75.5, 91.5, 107.5, 123.5, 139.5, 155.5];
      for (let i = 0; i < ringBoundaries.length; i++) {
        if (distanceMM <= ringBoundaries[i]) {
          return 10 - i;
        }
      }
      return 0;
    }
  </script>
</body>
</html>
