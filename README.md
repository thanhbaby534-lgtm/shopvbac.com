# SHOP
// app.js
const express = require('express');
const bodyParser = require('body-parser');
const crypto = require('crypto');

const app = express();
app.use(bodyParser.json());

const LOOT_PRICE = 50000; // 50k VND

const lootBoxConfig = {
  items: [
    { id: 1, accVip: "Vật phẩm Thường A", rarity: "Common", dropRate: 60 },
    { id: 2, accVip: "Vật phẩm Thường B", rarity: "Common", dropRate: 20 },
    { id: 3, accVip: "Vật phẩm Hiếm C", rarity: "Rare", dropRate: 15 },
    { id: 4, name: "Vật phẩm Huyền thoại D", rarity: "Legendary", dropRate: 5 }
  ]
};

// In-memory user store (demo). username/password mặc định "0000"
const users = {
  "0000": {
    username: "0000",
    password: "0000",
    balance: 150000, // ví dụ ban đầu 150k
    inventory: []
  }
};

// In-memory sessions: token -> username
const sessions = new Map();

// Helper: chọn item theo trọng số (hỗ trợ tổng != 100 và số thực)
function pickItemByWeight(items) {
  const total = items.reduce((s, it) => s + (Number(it.dropRate) || 0), 0);
  if (total <= 0) return null;
  // để tránh float issues, scale lên integer
  const scale = 1000000;
  const totalInt = Math.floor(total * scale);
  const r = crypto.randomInt(0, totalInt); // [0, totalInt)
  let cumulative = 0;
  for (const it of items) {
    cumulative += Math.floor((Number(it.dropRate) || 0) * scale);
    if (r < cumulative) return it;
  }
  return items[items.length - 1];
}

// Đăng nhập (demo): body { username, password } -> { token }
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  const user = users[username];
  if (!user || user.password !== password) {
    return res.status(401).json({ success: false, message: 'Đăng nhập thất bại' });
  }
  const token = crypto.randomUUID();
  sessions.set(token, username);
  return res.json({ success: true, token });
});

// Middleware auth
function authMiddleware(req, res, next) {
  const token = req.headers['authorization'];
  if (!token) return res.status(401).json({ success: false, message: 'Thiếu token' });
  const username = sessions.get(token);
  if (!username) return res.status(401).json({ success: false, message: 'Token không hợp lệ' });
  req.user = users[username];
  next();
}

// Endpoint mở túi mù
app.post('/open', authMiddleware, (req, res) => {
  const user = req.user;
  if (user.balance < LOOT_PRICE) {
    return res.status(400).json({ success: false, message: 'Số dư không đủ' });
  }

  // Simple in-memory atomicity: lock per user (demo). For production: DB transaction or distributed lock.
  if (user._locked) return res.status(429).json({ success: false, message: 'Đang xử lý yêu cầu khác, thử lại' });
  user._locked = true;

  try {
    const item = pickItemByWeight(lootBoxConfig.items);
    if (!item) {
      return res.status(500).json({ success: false, message: 'Lỗi cấu hình vật phẩm' });
    }

    user.balance -= LOOT_PRICE;
    user.inventory.push({
      id: item.id,
      name: item.name,
      rarity: item.rarity,
      obtainedAt: new Date().toISOString()
    });

    return res.json({
      success: true,
      message: 'Chúc mừng bạn!',
      itemReceived: item,
      newBalance: user.balance
    });
  } finally {
    user._locked = false;
  }
});

// Endpoint xem thông tin user (demo)
app.get('/me', authMiddleware, (req, res) => {
  const u = req.user;
  res.json({
    username: u.username,
    balance: u.balance,
    inventory: u.inventory
  });
});

const PORT = 3000;
app.listen(PORT, () => console.log(`Demo server chạy tại http://localhost:${PORT}`));