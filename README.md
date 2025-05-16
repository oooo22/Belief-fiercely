<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="utf-8">
  <title>Loop</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="https://fonts.googleapis.com/css2?family=Kanit&display=swap" rel="stylesheet">
  <style>
    body {
      font-family: 'Kanit', sans-serif;
      background: linear-gradient(135deg, #f9f3ff, #fffbe6);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 40px;
      min-height: 100vh;
      margin: 0;
    }

    h2 {
      font-size: 28px;
      color: #6a4fb3;
      margin-bottom: 10px;
    }

    p {
      font-size: 18px;
      color: #8e7cc3;
      margin-bottom: 30px;
    }

    .button-group {
      display: flex;
      gap: 20px;
      flex-wrap: wrap;
    }

    button {
      font-size: 18px;
      padding: 12px 24px;
      border: none;
      border-radius: 14px;
      background: linear-gradient(145deg, #f5d9ff, #ffeec9);
      color: #4b3b7c;
      transition: all 0.2s ease;
      cursor: pointer;
      box-shadow: 0 4px 10px rgba(0,0,0,0.1);
    }

    button:hover {
      background: linear-gradient(145deg, #ead0ff, #fff0b2);
      transform: scale(1.05);
    }

    button:active {
      transform: scale(0.97);
    }

    @media (max-width: 500px) {
      button {
        width: 100%;
      }
      .button-group {
        flex-direction: column;
        gap: 12px;
        width: 100%;
        max-width: 300px;
      }
    }
  </style>
</head>
<body>
  <h2>LOOP 🎧</h2>
  <p>กดค้างเพื่อพูด</p>

  <div class="button-group">
    <button id="startBtn">🎤 START</button>
    <button id="stopBtn">🛑 STOP</button>
  </div>

  <script>
    let audioCtx;
    let micStream;
    let micSource;
    let recorder;
    let recordedChunks = [];
    let loopSource;
    let loopInterval;

    let delayNodes = [];
    let feedbackGains = [];

    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');

    async function initAudio() {
      if (!audioCtx) {
        audioCtx = new AudioContext();

        const delayTimes = [0.8, 1.2]; // ลดการซ้อน & ชะลอ
        delayNodes = [];
        feedbackGains = [];

        let previousNode = null;

        delayTimes.forEach((t, i) => {
          const delay = audioCtx.createDelay(5.0);
          delay.delayTime.value = t;

          const feedback = audioCtx.createGain();
          feedback.gain.value = 0.35 + i * 0.05;

          delay.connect(feedback);
          feedback.connect(delay);

          delayNodes.push(delay);
          feedbackGains.push(feedback);

          if (previousNode) previousNode.connect(delay);
          previousNode = delay;
        });

        delayNodes[delayNodes.length - 1].connect(audioCtx.destination);

        loopInterval = setInterval(() => {
          feedbackGains.forEach(gain => {
            if (gain.gain.value < 0.6) {
              gain.gain.value += 0.01;
            }
          });
        }, 5000);
      }
    }

    async function startLoopPlayback(audioBuffer) {
      if (!audioCtx) return;

      loopSource = audioCtx.createBufferSource();
      loopSource.buffer = audioBuffer;
      loopSource.loop = true;

      const loopGain = audioCtx.createGain();
      loopGain.gain.value = 1.0;

      loopSource.connect(delayNodes[0]);
      loopSource.connect(loopGain).connect(audioCtx.destination);
      loopSource.start(0);
    }

    startBtn.addEventListener('pointerdown', async () => {
      if (startBtn._pressed) return;
      startBtn._pressed = true;

      await initAudio();

      try {
        micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        micSource = audioCtx.createMediaStreamSource(micStream);

        recorder = new MediaRecorder(micStream);
        recordedChunks = [];

        recorder.ondataavailable = (e) => {
          if (e.data.size > 0) recordedChunks.push(e.data);
        };

        recorder.onstop = async () => {
          const blob = new Blob(recordedChunks);
          const arrayBuffer = await blob.arrayBuffer();
          const audioBuffer = await audioCtx.decodeAudioData(arrayBuffer);
          startLoopPlayback(audioBuffer);
        };

        recorder.start();
        console.log("🎤 เริ่มอัดเสียง");
      } catch (err) {
        alert("ไม่สามารถใช้ไมโครโฟนได้: " + err.message);
      }
    });

    startBtn.addEventListener('pointerup', () => {
      if (!startBtn._pressed) return;
      startBtn._pressed = false;

      if (recorder && recorder.state === "recording") {
        recorder.stop();
        if (micStream) {
          micStream.getTracks().forEach(track => track.stop());
          micStream = null;
        }
        console.log("📴 หยุดอัดเสียง");
      }
    });

    stopBtn.onclick = () => {
      if (micStream) {
        micStream.getTracks().forEach(track => track.stop());
        micStream = null;
      }

      if (recorder && recorder.state === "recording") {
        recorder.stop();
      }

      if (loopSource) {
        loopSource.stop();
        loopSource.disconnect();
        loopSource = null;
      }

      if (loopInterval) clearInterval(loopInterval);

      if (audioCtx) {
        audioCtx.close();
        audioCtx = null;
      }

      console.log("🛑 หยุดเสียงทั้งหมด");
    };
  </script>
</body>
</html>
