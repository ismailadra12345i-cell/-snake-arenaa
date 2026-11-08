const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.static(path.join(__dirname, 'public')));

const ARENA_SIZE = 2000;
const players = {};
let foods = [];

for (let i = 0; i < 150; i++) {
  foods.push({
    id: Math.random().toString(36).substr(2, 9),
    x: Math.random() * ARENA_SIZE,
    y: Math.random() * ARENA_SIZE,
    color: `hsl(${Math.random() * 360}, 100%, 50%)`
  });
}

io.on('connection', (socket) => {
  socket.on('joinGame', ({ name, bet }) => {
    const startX = Math.random() * (ARENA_SIZE - 400) + 200;
    const startY = Math.random() * (ARENA_SIZE - 400) + 200;
    const segments = [];
    for (let i = 0; i < 15; i++) {
      segments.push({ x: startX - i * 8, y: startY });
    }
    players[socket.id] = {
      id: socket.id,
      name: name || 'لاعب',
      stake: Number(bet) || 100,
      angle: 0,
      speed: 4,
      color: `hsl(${Math.random() * 360}, 85%, 55%)`,
      segments,
      cashingOut: false,
      cashOutTimer: 0
    };
  });

  socket.on('updateAngle', (angle) => {
    if (players[socket.id] && !players[socket.id].cashingOut) {
      players[socket.id].angle = angle;
    }
  });

  socket.on('startCashOut', () => {
    if (players[socket.id]) {
      players[socket.id].cashingOut = true;
      players[socket.id].cashOutTimer = 3000;
    }
  });

  socket.on('disconnect', () => {
    delete players[socket.id];
  });
});

setInterval(() => {
  const playerIds = Object.keys(players);

  for (const id of playerIds) {
    const p = players[id];
    if (!p) continue;

    if (p.cashingOut) {
      p.cashOutTimer -= 33;
      if (p.cashOutTimer <= 0) {
        io.to(id).emit('cashedOut', p.stake);
        delete players[id];
        continue;
      }
    } else {
      const head = p.segments;
      const newHead = {
        x: Math.max(10, Math.min(ARENA_SIZE - 10, head.x + Math.cos(p.angle) * p.speed)),
        y: Math.max(10, Math.min(ARENA_SIZE - 10, head.y + Math.sin(p.angle) * p.speed))
      };
      p.segments.unshift(newHead);
      p.segments.pop();

      for (let i = foods.length - 1; i >= 0; i--) {
        const f = foods[i];
        if (Math.hypot(newHead.x - f.x, newHead.y - f.y) < 18) {
          foods.splice(i, 1);
          foods.push({
            id: Math.random().toString(36).substr(2, 9),
            x: Math.random() * ARENA_SIZE,
            y: Math.random() * ARENA_SIZE,
            color: `hsl(${Math.random() * 360}, 100%, 50%)`
          });
          const tail = p.segments[p.segments.length - 1];
          p.segments.push({ ...tail });
        }
      }
    }
  }

  for (const id1 of playerIds) {
    const p1 = players[id1];
    if (!p1) continue;
    const head1 = p1.segments;

    for (const id2 of playerIds) {
      if (id1 === id2) continue;
      const p2 = players[id2];
      if (!p2) continue;

      for (let i = 1; i < p2.segments.length; i++) {
        const seg = p2.segments[i];
        if (Math.hypot(head1.x - seg.x, head1.y - seg.y) < 14) {
          p2.stake += p1.stake;
          for (let k = 0; k < 10; k++) {
            const tail = p2.segments[p2.segments.length - 1];
            p2.segments.push({ ...tail });
          }
          io.to(id1).emit('eliminated');
          delete players[id1];
          break;
        }
      }
      if (!players[id1]) break;
    }
  }

  io.emit('state', { players, foods, arenaSize: ARENA_SIZE });
}, 33);

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log('Server running on port', PORT));
