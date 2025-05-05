// server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.static('public'));

const players = {};
const teams = {};

io.on('connection', socket => {
  console.log('Novo jogador conectado:', socket.id);

  const team = Object.values(teams).filter(t => t === 'greek').length <= Object.values(teams).filter(t => t === 'trojan').length ? 'greek' : 'trojan';
  const role = Math.random() < 0.5 ? 'warrior' : 'archer';
  teams[socket.id] = team;

  players[socket.id] = { x: 100, y: 100, hidden: false, team, role, hp: 100 };

  socket.emit('init', { players, id: socket.id });
  socket.broadcast.emit('new-player', { id: socket.id, data: players[socket.id] });

  socket.on('move', dir => {
    const speed = 5;
    const player = players[socket.id];
    if (!player) return;

    if (dir === 'left') player.x -= speed;
    if (dir === 'right') player.x += speed;
    if (dir === 'up') player.y -= speed;
    if (dir === 'down') player.y += speed;

    const inHorse = player.x > 250 && player.x < 350 && player.y > 150 && player.y < 250;
    player.hidden = (inHorse && player.team === 'greek');

    io.emit('update', { id: socket.id, data: player });
  });

  socket.on('attack', () => {
    const attacker = players[socket.id];
    if (!attacker) return;

    for (const id in players) {
      if (id === socket.id) continue;
      const target = players[id];

      const dx = target.x - attacker.x;
      const dy = target.y - attacker.y;
      const distance = Math.sqrt(dx * dx + dy * dy);

      const range = attacker.role === 'archer' ? 150 : 40;
      const damage = attacker.role === 'archer' ? 15 : 30;

      if (distance < range && attacker.team !== target.team) {
        target.hp -= damage;
        if (target.hp <= 0) {
          delete players[id];
          delete teams[id];
          io.emit('remove-player', id);
        } else {
          io.emit('update', { id, data: target });
        }
      }
    }
  });

  socket.on('disconnect', () => {
    delete players[socket.id];
    delete teams[socket.id];
    io.emit('remove-player', socket.id);
  });
});

server.listen(3000, () => {
  console.log('Servidor rodando em http://localhost:3000');
});

// public/index.html
/*
<!DOCTYPE html>
<html>
<head>
  <title>Cavalo de Troia</title>
  <style>
    body { margin: 0; background: #333; }
    canvas { display: block; margin: auto; background: #fdf6e3; border: 1px solid #555; }
  </style>
</head>
<body>
  <canvas id="game" width="600" height="400"></canvas>
  <p style="text-align:center; color:white;">Use as setas para mover, barra de espaço para atacar</p>
  <script src="/socket.io/socket.io.js"></script>
  <script src="script.js"></script>
</body>
</html>
*/

// public/script.js
/*
const socket = io();
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

const players = {};
let myId = null;

socket.on('init', data => {
  Object.assign(players, data.players);
  myId = data.id;
});

socket.on('new-player', ({ id, data }) => {
  players[id] = data;
});

socket.on('update', ({ id, data }) => {
  players[id] = data;
});

socket.on('remove-player', id => {
  delete players[id];
});

document.addEventListener('keydown', e => {
  if (e.key === 'ArrowLeft') socket.emit('move', 'left');
  if (e.key === 'ArrowRight') socket.emit('move', 'right');
  if (e.key === 'ArrowUp') socket.emit('move', 'up');
  if (e.key === 'ArrowDown') socket.emit('move', 'down');
  if (e.key === ' ') socket.emit('attack');
});

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = 'saddlebrown';
  ctx.fillRect(250, 150, 100, 100);
  ctx.strokeStyle = '#000';
  ctx.strokeRect(250, 150, 100, 100);

  for (const id in players) {
    const p = players[id];
    if (p.hidden && myId !== id && players[myId]?.team === 'trojan') continue;

    ctx.fillStyle = p.team === 'greek' ? 'blue' : 'red';
    ctx.fillRect(p.x, p.y, 30, 30);

    ctx.fillStyle = 'black';
    ctx.font = '12px Arial';
    ctx.fillText(`${p.role} (${p.hp})`, p.x, p.y - 5);
  }

  requestAnimationFrame(draw);
}

draw();
*/

// package.json
/*
{
  "name": "cavalo-de-troia",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "socket.io": "^4.7.2"
  }
}
*/
