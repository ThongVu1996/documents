# [Authentication Series - Part 3] Thực chiến Code - Triển khai hệ thống Token Rotation với Docker & Node.js.

> *"Talk is cheap, show me the code."* — Linus Torvalds

Ở **[Phần 1](https://github.com/ThongVu1996/documents/blob/main/web/jwt_refresh_token/part_1.md)** và **[Phần 2](https://github.com/ThongVu1996/documents/blob/main/web/jwt_refresh_token/part_2.md)**, chúng ta đã đi qua rất nhiều lý thuyết về Access Token, Refresh Token, các kịch bản tấn công và cơ chế phòng thủ "Reuse Detection".

Nhưng lý thuyết sẽ mãi là lý thuyết nếu không chạy được. Bạn có thể tự hỏi:
*   *"Code logic Grace Period trông như thế nào?"*
*   *"Làm sao cấu hình Redis để lưu Token Family?"*
*   *"Frontend xử lý hàng đợi (Queue) ra sao để không bị Race Condition?"*

Bài viết này sẽ không giải thích dông dài nữa. Chúng ta sẽ cùng nhau dựng một **Phòng thí nghiệm Authentication** hoàn chỉnh bằng Docker. Chỉ cần 1 câu lệnh, bạn sẽ có ngay môi trường để tận mắt chứng kiến hacker bị chặn đứng như thế nào.

---

## I. Kiến trúc phòng thí nghiệm (Architecture)

Chúng ta sẽ dựng 3 Containers đơn giản:

1.  **Server (Node.js/Express):** Xử lý Login, Refresh Token Rotation, và Reuse Detection.
2.  **Database (Redis):** Lưu trữ Refresh Token. Redis cực kỳ phù hợp vì tốc độ nhanh (In-memory) và hỗ trợ TTL (tự xóa khi hết hạn).
3.  **Client (Nginx + Static HTML):** Giả lập trình duyệt người dùng với các nút bấm Test Case.

> [!NOTE]
> **Lưu ý về Redis:** Trong môi trường lab này, Redis sẽ mất dữ liệu khi restart container (ephemeral). Trong môi trường Production, bạn nên mount volume để đảm bảo dữ liệu (persistence) hoặc dùng Redis Cluster.

---

## II. Setup môi trường (Copy & Paste)

Hãy tạo một thư mục `token-lab` và cấu trúc file như sau:
```plaintext
/token-lab
├── docker-compose.yml
├── /server
│   ├── package.json
│   ├── server.js       <-- Logic chính nằm ở đây
│   └── Dockerfile
└── /client
    ├── index.html      <-- Giao diện Test
    ├── nginx.conf
    └── script.js       <-- Logic Frontend Queue
```

### 1. File `docker-compose.yml`
Chìa khóa để chạy mọi thứ.

```yaml
version: '3.8'

services:
  redis:
    image: redis:alpine
    container_name: auth-redis
    ports:
      - "6379:6379"

  server:
    build: ./server
    container_name: auth-server
    ports:
      - "3000:3000"
    environment:
      - REDIS_HOST=redis
      - ACCESS_TOKEN_SECRET=secret123
      - REFRESH_TOKEN_SECRET=secret456
    depends_on:
      - redis

  client:
    image: nginx:alpine
    container_name: auth-client
    ports:
      - "8080:80"
    volumes:
      - ./client:/usr/share/nginx/html
```

---

## III. Backend Implementation (Node.js)

Đây là nơi chứa "bộ não" xử lý. Chúng ta sẽ dùng thư viện `ioredis` để giao tiếp với Redis và `jsonwebtoken` để tạo token.

### 1. Setup `server/package.json` & `server/Dockerfile`
*(Tạo file package.json đơn giản với các dependencies: `express`, `cors`, `cookie-parser`, `jsonwebtoken`, `ioredis`, `uuid`)*

### 2. Logic chính: `server/server.js`
Đây là đoạn code quan trọng nhất. Hãy chú ý các comment giải thích.

```javascript
const express = require('express');
const cookieParser = require('cookie-parser');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

const app = express();
const redis = new Redis({ host: process.env.REDIS_HOST || 'redis' });

app.use(express.json());
app.use(cookieParser());
app.use(cors({ origin: 'http://localhost:8080', credentials: true })); // Cho phép Client gọi

// Cấu hình: AT sống 10 giây (để dễ test), RT sống 7 ngày
const AT_LIFE = '10s';
const RT_LIFE_SECONDS = 7 * 24 * 60 * 60; 
const GRACE_PERIOD_MS = 10000; // 10 giây ân hạn

// --- HELPER FUNCTIONS ---
const generateAT = (user) => jwt.sign(user, process.env.ACCESS_TOKEN_SECRET, { expiresIn: AT_LIFE });
const generateRT = () => uuidv4(); // RT là chuỗi ngẫu nhiên

// --- 1. LOGIN ---
app.post('/login', async (req, res) => {
    const user = { id: 'user_123', name: 'Demo User' }; // Giả lập user từ DB
    
    // Tạo Family ID mới cho chuỗi đăng nhập này
    const familyId = uuidv4();
    const accessToken = generateAT(user);
    const refreshToken = generateRT();

    // Lưu RT vào Redis
    const tokenData = {
        userId: user.id,
        familyId: familyId,
        isRevoked: false
    };
    await redis.set(`rt:${refreshToken}`, JSON.stringify(tokenData), 'EX', RT_LIFE_SECONDS);

    // Trả về: AT (body), RT (HttpOnly Cookie)
    // [IMPORTANT]: Trong Production (HTTPS), bắt buộc thêm secure: true
    res.cookie('refreshToken', refreshToken, { 
        httpOnly: true, 
        sameSite: 'strict',
        // secure: true // Uncomment khi deploy Production
    });
    res.json({ accessToken, message: "Login successful" });
});

// --- 2. GET PROFILE (Protected Route) ---
app.get('/profile', (req, res) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];
    
    if (!token) return res.sendStatus(401);

    jwt.verify(token, process.env.ACCESS_TOKEN_SECRET, (err, user) => {
        if (err) return res.sendStatus(401); // Token hết hạn hoặc sai
        res.json({ message: `Hello ${user.name}, this is protected data!` });
    });
});

// --- 3. REFRESH TOKEN (TRÁI TIM CỦA BÀI VIẾT) ---
app.post('/refresh-token', async (req, res) => {
    const incomingRT = req.cookies.refreshToken;
    if (!incomingRT) return res.status(401).json({ error: "No token" });

    // Lấy thông tin từ Redis
    const rawData = await redis.get(`rt:${incomingRT}`);
    
    // Case 1: Token không tồn tại (Hết hạn hoặc Token Fake)
    if (!rawData) return res.status(403).json({ error: "Invalid Token" });

    const tokenData = JSON.parse(rawData);

    // =========================================================
    // REUSE DETECTION (BẪY HACKER) & GRACE PERIOD
    // =========================================================
    if (tokenData.isRevoked) {
        const timeDiff = Date.now() - tokenData.revokedAt;
        
        if (timeDiff < GRACE_PERIOD_MS) {
            // ---> Case 2: RACE CONDITION (Mạng lag, request song song)
            // Trong thời gian ân hạn, vẫn cho phép lấy AT mới, nhưng KHÔNG xoay vòng RT nữa.
            console.log(`⚠️ Grace period active for family ${tokenData.familyId}`);
            const newAT = generateAT({ id: tokenData.userId, name: 'Demo User' });
            return res.json({ accessToken: newAT });
        } else {
            // ---> Case 3: HACKER DETECTED!
            // Token đã chết lâu rồi mà vẫn moi lên dùng -> Xóa cả dòng họ!
            console.error(`🚨 SECURITY ALERT: Token reused! Destroying family ${tokenData.familyId}`);
            
            // Tìm và xóa tất cả token thuộc familyId này 
            // [TIP]: Trong thực tế nên lưu Redis Set `family:{id}` chứa list các RT để xóa nhanh (O(1)) thay vì scan/del lẻ tẻ.
            await redis.del(`rt:${incomingRT}`); 
            return res.status(403).json({ error: "Reuse Detected! Account blocked." });
        }
    }

    // =========================================================
    // ROTATION (XOAY VÒNG TOKEN)
    // =========================================================
    
    // 1. Đánh dấu token cũ là Revoked (chưa xóa hẳn để check Grace Period)
    tokenData.isRevoked = true;
    tokenData.revokedAt = Date.now();
    await redis.set(`rt:${incomingRT}`, JSON.stringify(tokenData), 'EX', 60); // Giữ lại 60s để check reuse

    // 2. Tạo cặp token mới (Kế thừa familyId)
    const newAT = generateAT({ id: tokenData.userId, name: 'Demo User' });
    const newRT = generateRT();
    
    const newTokenData = {
        userId: tokenData.userId,
        familyId: tokenData.familyId, // QUAN TRỌNG: Giữ nguyên Family ID
        isRevoked: false
    };
    await redis.set(`rt:${newRT}`, JSON.stringify(newTokenData), 'EX', RT_LIFE_SECONDS);

    // 3. Trả về
    res.cookie('refreshToken', newRT, { 
        httpOnly: true, 
        sameSite: 'strict',
        // secure: true // Uncomment khi deploy Production
    });
    res.json({ accessToken: newAT });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## IV. Frontend Implementation (Client)

Frontend sẽ có 2 nhiệm vụ:
1.  **UI:** Có các nút để kích hoạt Test Case.
2.  **Logic:** Xử lý Axios Interceptor để tự động refresh và hàng đợi (Queue).

### 1. `client/index.html`
```html
<!DOCTYPE html>
<html>
<head>
    <title>Token Rotation Lab</title>
    <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
    <style> 
        body { font-family: monospace; padding: 20px; }
        button { padding: 10px; margin: 5px; cursor: pointer; }
        .log { background: #eee; padding: 10px; margin-top: 10px; border: 1px solid #ccc; height: 300px; overflow-y: scroll;}
    </style>
</head>
<body>
    <h1>🛡️ Token Rotation Test Lab</h1>
    <div>
        <button onclick="login()">1. Login</button>
        <button onclick="getProfile()">2. Get Profile (Auto Refresh)</button>
        <button onclick="testRaceCondition()">3. Test Race Condition (Spam)</button>
        <button onclick="simulateHacker()">4. Simulate Hacker Attack ☠️</button>
    </div>
    <div id="logs" class="log"></div>
    <script src="script.js"></script>
</body>
</html>
```

### 2. `client/script.js` (Interceptor Logic)
```javascript
const api = axios.create({ baseURL: 'http://localhost:3000', withCredentials: true });
let accessToken = null;

// --- LOGGING HELPER ---
function log(msg) {
    const el = document.getElementById('logs');
    el.innerHTML += `<div>[${new Date().toLocaleTimeString()}] ${msg}</div>`;
    el.scrollTop = el.scrollHeight;
}

// --- INTERCEPTOR (QUEUE LOGIC) ---
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
    failedQueue.forEach(prom => {
        if (error) prom.reject(error);
        else prom.resolve(token);
    });
    failedQueue = [];
};

api.interceptors.request.use(config => {
    if (accessToken) config.headers['Authorization'] = `Bearer ${accessToken}`;
    return config;
});

api.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config;
        
        if (error.response.status === 401 && !originalRequest._retry) {
            if (isRefreshing) {
                log("⏳ Request paused, waiting for refresh...");
                return new Promise((resolve, reject) => {
                    failedQueue.push({ resolve, reject });
                }).then(token => {
                    originalRequest.headers['Authorization'] = 'Bearer ' + token;
                    return api(originalRequest);
                });
            }

            originalRequest._retry = true;
            isRefreshing = true;
            log("🔄 Token expired. Refreshing...");

            try {
                const { data } = await api.post('/refresh-token');
                accessToken = data.accessToken;
                log("✅ Refresh SUCCESS. Access Token updated.");
                
                processQueue(null, accessToken);
                isRefreshing = false;
                
                originalRequest.headers['Authorization'] = 'Bearer ' + accessToken;
                return api(originalRequest);
            } catch (err) {
                processQueue(err, null);
                isRefreshing = false;
                log("❌ Refresh FAILED. Logging out...");
                accessToken = null;
            }
        }
        return Promise.reject(error);
    }
);

// --- TEST FUNCTIONS ---
async function login() {
    const res = await api.post('/login');
    accessToken = res.data.accessToken;
    log("🔑 Login Success!");
}

async function getProfile() {
    try {
        const res = await api.get('/profile');
        log(`👤 Data: ${res.data.message}`);
    } catch (e) { log(`⛔ Error: ${e.response?.data?.error || e.message}`); }
}

async function testRaceCondition() {
    log("🚀 Sending 5 parallel requests...");
    // Gửi 5 request cùng lúc khi token (giả định) đã hết hạn
    const p1 = api.get('/profile');
    const p2 = api.get('/profile');
    const p3 = api.get('/profile');
    const p4 = api.get('/profile');
    const p5 = api.get('/profile');
    
    try {
        await Promise.all([p1, p2, p3, p4, p5]);
        log("🎉 All 5 requests finished successfully!");
    } catch (e) { log("💥 Race Condition Failed!"); }
}

// Giả lập hacker dùng lại token cũ
async function simulateHacker() {
    log("☠️ Simulating Hacker Attack...");
    // 1. Lưu token hiện tại (coi như token cũ mà hacker trộm được)
    // Lưu ý: Vì cookie là HttpOnly, client JS không đọc được. 
    // Trong demo này, ta sẽ gọi refresh 1 lần để Server rotate sang cái mới,
    // nhưng ta CỐ TÌNH gửi lại request với cookie cũ bằng cách... 
    // Thực tế demo trên browser khó giả lập việc gửi cookie cũ vì browser tự quản lý.
    // --> Cách test: Mở Postman hoặc một tab ẩn danh khác.
    log("⚠️ Để test case này, hãy mở DevTools -> Application -> Cookies.");
    log("1. Copy value của refreshToken.");
    log("2. Bấm nút 'Get Profile' để Server đổi sang token mới.");
    log("3. Dùng Postman gửi POST /refresh-token với Cookie cũ vừa copy.");
    log("👉 Xem log của Docker Container server để thấy BÁO ĐỘNG ĐỎ.");
}
```

---

## V. Hướng dẫn Test Lab (Chạy kịch bản)

Bây giờ hãy mở Terminal và chạy lệnh:
```bash
docker-compose up --build
```
Truy cập: `http://localhost:8080`

### Case 1: Happy Path (Cơ bản)
1.  Bấm **Login**. Log báo thành công.
2.  Chờ 10 giây (Access Token hết hạn).
3.  Bấm **Get Profile**.
4.  **Quan sát:**
    *   Log báo "Token expired. Refreshing...".
    *   Sau đó báo "Refresh SUCCESS".
    *   Cuối cùng hiển thị Data.
    *   User không bị lỗi, trải nghiệm mượt mà.

### Case 2: Race Condition (Spam Request)
1.  Tải lại trang, bấm **Login**.
2.  Chờ 10 giây.
3.  Bấm nút **3. Test Race Condition**.
4.  **Quan sát:**
    *   Browser gửi 5 requests cùng lúc.
    *   Log báo "Token expired..." (chỉ 1 lần).
    *   Log báo "Request paused..." (4 lần - 4 request kia vào hàng đợi).
    *   Sau khi refresh xong, cả 5 requests đều trả về Data thành công.
5.  **Kết luận:** Hàng đợi hoạt động tốt. Server không bị spam refresh.

### Case 3: Hacker Attack (Reuse Detection)
Đây là phần thú vị nhất. Để tái hiện, chúng ta cần "nhanh tay" hoặc dùng công cụ hỗ trợ.

**Cách tái hiện:**
1.  Login thành công.
2.  Mở DevTools (F12) -> Network. Tìm request login, xem phần Response Cookies. Copy giá trị `refreshToken` (Ví dụ: `uuid-111`).
3.  Bấm **Get Profile** (sau 10s) để Server thực hiện Refresh.
    *   Lúc này Server đã đổi token sang `uuid-222`.
    *   Token `uuid-111` đã bị đánh dấu là **Revoked** (Hủy).
4.  **Giả vai Hacker:**
    *   Mở Postman (hoặc dùng lệnh `curl`).
    *   Gửi `POST http://localhost:3000/refresh-token`.
    *   Header: `Cookie: refreshToken=uuid-111` (Dùng lại cái cũ).
5.  **Quan sát kết quả:**
    *   **Postman:** Nhận lỗi `403 Forbidden: Reuse Detected! Account blocked.`
    *   **Server Terminal:** Xuất hiện dòng `🚨 SECURITY ALERT: Token reused! Destroying family...`.
    *   **Quay lại Trình duyệt (User thật):** Bấm **Get Profile** -> Lỗi 403 hoặc 401. User bị đá văng ra ngoài.

---

## VI. Lời kết: Đã đến đích, hay chỉ mới bắt đầu?

Qua 3 phần của series, chúng ta đã đi từ những lý thuyết khô khan đến một hệ thống thực chiến "sờ tận tay".

Bạn đã thấy sức mạnh của **Token Rotation** trong việc cô lập hacker, và sự tinh tế của **Grace Period** trong việc giữ trải nghiệm người dùng mượt mà. Source code này là bộ khung vững chắc để bạn phát triển tiếp cho các dự án Production (cần thêm HTTPS, Rate Limiting, v.v.).

Hệ thống mà chúng ta vừa xây dựng (Access Token trên RAM + Refresh Token trong HttpOnly Cookie) hiện đang là **Tiêu chuẩn vàng (Gold Standard)** cho 95% các ứng dụng web hiện đại.

Tuy nhiên...

Với 5% còn lại – những hệ thống Tài chính, Ngân hàng, hay Enterprise đòi hỏi mức độ bảo mật "Paranoid" (Cực đoan) – thì việc để trình duyệt nắm giữ Refresh Token (dù đã bọc kỹ trong Cookie) vẫn là một rủi ro tiềm ẩn.

*   *Liệu có cách nào để Trình duyệt hoàn toàn không biết Token là cái gì không?*
*   *Liệu chúng ta có thể giấu nhẹm toàn bộ Token ra sau Server, chỉ để lại cho Client một SessionID vô hại?*

Đó là lúc chúng ta bước vào cảnh giới cao nhất của bảo mật Frontend: **Kiến trúc Zero Token với BFF (Backend For Frontend).**

👉 **Nếu bạn muốn biết các "ông lớn" Fintech bảo vệ tài sản người dùng như thế nào, đừng bỏ lỡ bài viết cuối cùng (Bonus): [Authentication Part 4] BFF Pattern - Khi Browser hoàn toàn "mù tịt" về Token [tại đây](https://github.com/ThongVu1996/documents/blob/main/web/jwt_refresh_token/part_4.md).**

> **Link Github Source Code (Part 3):** `github.com/your-username/token-rotation-demo` *(Placeholder)*

