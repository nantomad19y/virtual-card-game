<!DOCTYPE html>
<html lang="lo">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Real-time Carnival Wheel</title>
  <script src="/socket.io/socket.io.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: sans-serif; }
    body { background: #121212; color: #fff; text-align: center; padding: 20px; }
    .status-bar { margin: 15px 0; font-size: 1.2rem; }
    .balance { color: #00e676; font-weight: bold; }
    .timer { color: #ff9800; font-weight: bold; font-size: 1.5rem; }
    
    .wheel-box { position: relative; width: 320px; height: 320px; margin: 20px auto; }
    #wheel { width: 100%; height: 100%; border-radius: 50%; border: 6px solid #ffd700; transition: transform 5s cubic-bezier(0.1, 0.9, 0.1, 1); }
    .pointer { position: absolute; top: -10px; left: 50%; transform: translateX(-50%); width: 0; height: 0; border-left: 12px solid transparent; border-right: 12px solid transparent; border-top: 22px solid #ff1744; z-index: 10; }

    .grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; max-width: 360px; margin: 15px auto; }
    .card { background: #222; border: 2px solid #444; padding: 15px; border-radius: 8px; cursor: pointer; }
    .card.active { border-color: #00e676; background: #1b3e20; }
    .num { font-size: 1.5rem; font-weight: bold; }
  </style>
</head>
<body>

  <h1>🎡 Carnival Treasure (Real-time)</h1>
  <div class="status-bar">
    ເງິນ: <span id="balance" class="balance">1000</span> 💰 | 
    ເວລາ: <span id="timer" class="timer">15</span>s
  </div>

  <div class="wheel-box">
    <div class="pointer"></div>
    <canvas id="wheel" width="320" height="320"></canvas>
  </div>

  <h3 id="msg">ກະລຸນາເລືອກເດີມພັນ...</h3>

  <div class="grid" id="grid"></div>

  <script>
    const socket = io();
    let balance = 1000;
    let selectedOption = null;
    let currentRotation = 0;
    let wheelList = [];

    const canvas = document.getElementById('wheel');
    const ctx = canvas.getContext('2d');

    socket.on('init-state', (data) => {
      wheelList = data.wheelList;
      drawWheel();
      buildGrid();
    });

    function drawWheel() {
      const total = wheelList.length;
      const angle = (2 * Math.PI) / total;
      for (let i = 0; i < total; i++) {
        ctx.beginPath();
        ctx.fillStyle = wheelList[i].color;
        ctx.moveTo(160, 160);
        ctx.arc(160, 160, 150, i * angle, (i + 1) * angle);
        ctx.fill();
        ctx.stroke();

        ctx.save();
        ctx.translate(160, 160);
        ctx.rotate(i * angle + angle / 2);
        ctx.textAlign = "right";
        ctx.fillStyle = "#fff";
        ctx.font = "bold 10px Arial";
        ctx.fillText(wheelList[i].label, 140, 4);
        ctx.restore();
      }
    }

    function buildGrid() {
      const opts = [
        { label: '1', mult: '1:1', color: '#2196f3' },
        { label: '2', mult: '2:1', color: '#4caf50' },
        { label: '4', mult: '4:1', color: '#ff9800' },
        { label: '7', mult: '7:1', color: '#9c27b0' },
        { label: '18', mult: '18:1', color: '#e91e63' },
        { label: '40', mult: '40:1', color: '#f44336' }
      ];
      const grid = document.getElementById('grid');
      grid.innerHTML = '';
      opts.forEach(o => {
        const card = document.createElement('div');
        card.className = 'card';
        card.innerHTML = `<div class="num" style="color:${o.color}">${o.label}</div><div>${o.mult}</div>`;
        card.onclick = () => {
          if (balance < 50) return alert('ເງິນບໍ່ພໍ!');
          document.querySelectorAll('.card').forEach(c => c.classList.remove('active'));
          card.classList.add('active');
          selectedOption = o.label;
          socket.emit('place-bet', { label: o.label, amount: 50 });
        };
        grid.appendChild(card);
      });
    }

    socket.on('timer-update', (t) => {
      document.getElementById('timer').innerText = t;
      if (t === 0) document.getElementById('msg').innerText = "ປິດຮັບເດີມພັນ! ວົງລໍ້ກຳລັງໝຸນ...";
    });

    socket.on('start-spin', (data) => {
      const total = wheelList.length;
      const segAngle = 360 / total;
      const targetAngle = (total - data.winningIndex) * segAngle - (segAngle / 2);
      
      currentRotation += (360 * 5) + targetAngle - (currentRotation % 360);
      canvas.style.transform = `rotate(${currentRotation}deg)`;
    });

    socket.on('bet-result', (res) => {
      if (res.win) {
        balance += res.amount;
        document.getElementById('msg').innerText = `🎉 ຍິນດີດ້ວຍ! ທ່ານໄດ້ຮັບ +${res.amount} 💰`;
      } else {
        balance -= 50;
        document.getElementById('msg').innerText = `❌ ເສຍໃຈດ້ວຍ!`;
      }
      document.getElementById('balance').innerText = balance;
    });

    socket.on('reset-game', (t) => {
      document.getElementById('msg').innerText = "ເລີ່ມຕາໃໝ່! ກະລຸນາເລືອກເດີມພັນ...";
      document.querySelectorAll('.card').forEach(c => c.classList.remove('active'));
      selectedOption = null;
    });
  </script>
</body>
</html>
