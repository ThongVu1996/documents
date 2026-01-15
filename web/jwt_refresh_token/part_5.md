# [Authentication Series - Part 5] Thực chiến BFF: Xây dựng Security Proxy "One-Click" với Next.js, Laravel & Redis

> "Talk is cheap, show me the code." — Linus Torvalds

Ở [Phần 4](./part_4.md), chúng ta đã thiết kế kiến trúc **"Zero Token Architecture"** để đạt bảo mật cấp ngân hàng. Mục tiêu tối thượng là: **Trình duyệt chỉ giữ SessionID vô hại, còn Access/Refresh Token được giấu kín hoàn toàn ở Server (BFF).**

Hôm nay, chúng ta sẽ hiện thực hóa nó. Hệ thống bao gồm:
*   **BFF (Next.js):** Đóng vai trò "Vệ sĩ", quản lý Session và tự động Refresh Token.
*   **Core API (Laravel):** Xử lý nghiệp vụ, xác thực JWT.
*   **Redis:** "Két sắt" lưu trữ Token.
*   **Docker Automation:** Script tự động cài đặt để chỉ cần 1 lệnh khởi chạy.

---

## I. Kiến trúc hệ thống (Architecture)

1.  **Client (Browser):** Next.js Frontend. Chỉ lưu **Session Cookie (HttpOnly)**.
2.  **BFF (Next.js API Routes):** Middleware trung gian.
    *   Nhận SessionID -> Tra cứu Redis -> Lấy JWT.
    *   Proxy request sang Laravel.
    *   **Smart Refresh:** Tự động Refresh Token khi gặp lỗi 401 (xử lý Race Condition).
3.  **Backend Core (Laravel):** API Service chứa logic nghiệp vụ và cấp JWT.

---

## II. Tại sao lại là Redis? (The Session Store)

Trước khi code, cần hiểu rõ vai trò sống còn của Redis trong mô hình này.

### 1. Vai trò
Hãy tưởng tượng Redis là một **"Chiếc tủ gửi đồ thông minh"**:
*   **Browser:** Chỉ giữ cái thẻ số (SessionID).
*   **Redis:** Là cái tủ chứa đồ thật (Access Token, Refresh Token).
*   **Next.js (BFF):** Là người bảo vệ cầm chìa khóa tủ.

### 2. Tại sao cần Redis?
*   **Tốc độ (In-Memory):** Mọi request từ Client đều phải tra cứu Session. Redis phản hồi trong micro-seconds, nhanh gấp bội so với SQL.
*   **Tự hủy (TTL):** Chúng ta set hạn 7 ngày cho Token trong Redis. Hết 7 ngày, Redis tự xóa. Không cần chạy Cronjob dọn rác.
*   **Chia sẻ trạng thái (Scalability):** Nếu bạn chạy 5 server Next.js, tất cả đều nhìn thấy chung một kho Session trong Redis.

---

## III. Setup dự án & Automation (The "One-Click" Run)

Chúng ta sẽ sử dụng cấu trúc thư mục Monorepo:
```plaintext
/ultimate-auth
├── docker-compose.yml
├── /backend                 <-- Laravel 10/11
│   ├── Dockerfile
│   ├── entrypoint.sh        <-- Script tự động cài đặt Backend
│   ├── composer.json
│   └── ... (Source code Laravel)
└── /frontend                <-- Next.js App Router
    ├── Dockerfile
    ├── entrypoint.sh        <-- Script tự động cài đặt Frontend
    ├── package.json
    └── ... (Source code Next.js)
```

### 1. File kết nối: `docker-compose.yml`
Tạo file này ở thư mục gốc `ultimate-auth/`.

```yaml
version: '3.8'

services:
  # 1. Database cho Token & Session
  redis:
    image: redis:alpine
    container_name: auth-redis
    ports: ["6379:6379"]

  # 2. Backend Core (Laravel API)
  backend:
    build: ./backend
    container_name: auth-backend
    ports: ["8000:8000"]
    volumes:
      - ./backend:/var/www
    environment:
      - DB_CONNECTION=sqlite
    depends_on:
      - redis

  # 3. BFF & Frontend (Next.js)
  frontend:
    build: ./frontend
    container_name: auth-frontend
    ports: ["3000:3000"]
    volumes:
      - ./frontend:/app
      - /app/node_modules # Tránh ghi đè node_modules của container
    environment:
      - REDIS_HOST=redis
      - BACKEND_URL=http://backend:8000/api
    depends_on:
      - backend
      - redis
```

---

## IV. Triển khai Backend (Laravel Automation)

Trong thư mục `backend/`, chúng ta cần config Docker để nó tự động cài Laravel, Database và JWT.

### 1. `backend/Dockerfile`
```dockerfile
FROM php:8.2-fpm

# Cài đặt dependencies và SQLite
RUN apt-get update && apt-get install -y \
    git unzip libzip-dev sqlite3 libsqlite3-dev

# Cài PHP Extensions
RUN docker-php-ext-install pdo pdo_sqlite zip

# Cài Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

# Copy Script tự động hóa
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

### 2. `backend/entrypoint.sh` (Magic Script)
Script này thay thế hoàn toàn các thao tác thủ công.

```bash
#!/bin/bash
set -e

echo "🚀 Starting Backend Setup..."

# 1. Cài đặt Composer Dependencies
if [ ! -d "vendor" ]; then
    echo "📦 Installing Composer Dependencies..."
    composer install --no-interaction --optimize-autoloader
else
    echo "✅ Vendor exists, skipping install."
fi

# 2. Setup .env và cấu hình SQLite
if [ ! -f ".env" ]; then
    echo "📄 Creating .env file..."
    cp .env.example .env
    # Sửa config DB sang SQLite tự động
    sed -i 's/DB_CONNECTION=mysql/DB_CONNECTION=sqlite/g' .env
    sed -i 's/DB_HOST=127.0.0.1/#DB_HOST=127.0.0.1/g' .env
    sed -i 's/DB_DATABASE=laravel/DB_DATABASE=\/var\/www\/database\/database.sqlite/g' .env
fi

# 3. Tạo Database File
if [ ! -f "database/database.sqlite" ]; then
    echo "🗄️ Creating SQLite database..."
    touch database/database.sqlite
fi

# 4. Sinh Key và Secret
if ! grep -q "APP_KEY=base64" .env; then
    php artisan key:generate
fi
if ! grep -q "JWT_SECRET=" .env; then
    echo "🔐 Generating JWT Secret..."
    php artisan jwt:secret --force
fi

# 5. Chạy Migration
echo "🔄 Running Migrations..."
php artisan migrate --force

echo "✅ Backend Ready! Starting Server..."
exec php artisan serve --host=0.0.0.0 --port=8000
```

### 3. Logic Code Laravel (`AuthController.php`)
*(Yêu cầu: Đã cài `tymon/jwt-auth`)*

```php
<?php

namespace App\Http\Controllers;
use App\Http\Controllers\Controller;

class AuthController extends Controller
{
    public function __construct() {
        $this->middleware('auth:api', ['except' => ['login', 'refresh']]);
    }

    public function login() {
        $credentials = request(['email', 'password']);
        if (! $token = auth()->attempt($credentials)) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        return $this->respondWithToken($token);
    }

    public function me() {
        return response()->json(auth()->user());
    }

    public function refresh() {
        return $this->respondWithToken(auth()->refresh());
    }

    protected function respondWithToken($token) {
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => auth()->factory()->getTTL() * 60,
            'refresh_token' => $token // Giả lập refresh token
        ]);
    }
}
```

---

## V. Triển khai BFF (Next.js Automation)
Trong thư mục `frontend/`.

### 1. `frontend/Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

### 2. `frontend/entrypoint.sh`
```bash
#!/bin/sh
set -e
echo "🚀 Starting Frontend Setup..."

if [ ! -d "node_modules" ]; then
    echo "📦 Installing NPM Dependencies..."
    npm install
fi

echo "✅ Frontend Ready! Starting Next.js..."
exec npm run dev
```

### 3. Cấu hình Redis (`frontend/lib/redis.ts`)
```typescript
import Redis from 'ioredis';
// 'redis' là tên host trong mạng Docker
const redis = new Redis({ host: process.env.REDIS_HOST || 'redis', port: 6379 });
export default redis;
```

---

## VI. Core Feature 1: Đăng nhập & Tạo Session
Đây là cổng vào duy nhất. Biến đổi User/Pass thành SessionID.

**File:** `frontend/app/api/auth/login/route.ts`

```typescript
import { NextResponse } from 'next/server';
import { v4 as uuidv4 } from 'uuid';
import redis from '@/lib/redis';
import axios from 'axios';

const BACKEND_URL = process.env.BACKEND_URL;

export async function POST(request: Request) {
  try {
    const body = await request.json();

    // 1. Gọi Laravel để lấy cặp Token
    const { data: tokens } = await axios.post(`${BACKEND_URL}/auth/login`, body);

    // 2. Sinh SessionID (Chuỗi ngẫu nhiên vô hại)
    const sessionId = uuidv4();

    // 3. Lưu Token thật vào Redis (Giấu tiệt ở Server)
    // Key: session:{uuid} -> Value: JSON Token
    await redis.set(`session:${sessionId}`, JSON.stringify(tokens), 'EX', 7 * 86400);

    // 4. Trả về Cookie cho Browser
    const response = NextResponse.json({ success: true, user: tokens.user });
    
    response.cookies.set({
      name: 'session_id',
      value: sessionId,
      httpOnly: true, // JS không đọc được
      secure: false,  // Dev mode
      sameSite: 'lax',
      path: '/',
      maxAge: 7 * 86400,
    });

    return response;
  } catch (error) {
    return NextResponse.json({ error: 'Login Failed' }, { status: 401 });
  }
}
```

---

## VII. Core Feature 2 & 3: Proxy & Smart Refresh
Chúng ta cần tạo một Proxy Client thông minh để hứng request và xử lý Race Condition.

### 1. Proxy Client Logic (`frontend/lib/proxy.ts`)
Đây là nơi chứa "ma thuật" Promise Singleton.

> [!WARNING]
> Lưu ý: Logic `let refreshPromise` dưới đây chỉ hoạt động trên 1 instance Next.js. Nếu scale nhiều server, bạn cần dùng **Distributed Lock** (Redlock) trên Redis.

```typescript
import axios from 'axios';
import redis from '@/lib/redis';

const BACKEND_URL = process.env.BACKEND_URL;

// Biến Singleton giữ Promise Refresh (Cơ chế khóa Local)
let refreshPromise: Promise<string> | null = null;

export const createProxyClient = (sessionId: string) => {
  const client = axios.create({ baseURL: BACKEND_URL });

  // --- 1. Hydrate Token: Lấy Token từ Redis gắn vào Header ---
  client.interceptors.request.use(async (config) => {
    const rawData = await redis.get(`session:${sessionId}`);
    if (rawData) {
      const { access_token } = JSON.parse(rawData);
      config.headers.Authorization = `Bearer ${access_token}`;
    }
    return config;
  });

  // --- 2. Smart Refresh: Xử lý lỗi 401 & Race Condition ---
  client.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config;

      // Nếu gặp 401 và chưa retry lần nào
      if (error.response?.status === 401 && !originalRequest._retry) {
        originalRequest._retry = true;

        // --- BẮT ĐẦU LOGIC QUEUE ---
        if (!refreshPromise) {
            console.log(`🔄 [BFF] Token expired. Calling Laravel Refresh...`);
            
            // Tạo khóa Promise
            refreshPromise = (async () => {
                const rawData = await redis.get(`session:${sessionId}`);
                if (!rawData) throw new Error('No session');
                const { access_token } = JSON.parse(rawData);

                // Gọi Laravel Endpoint Refresh
                const { data: newTokens } = await axios.post(
                    `${BACKEND_URL}/auth/refresh`, {}, 
                    { headers: { Authorization: `Bearer ${access_token}` }}
                );

                // Cập nhật Redis với Token mới (Giữ nguyên TTL cũ)
                const updatedData = { ...JSON.parse(rawData), ...newTokens };
                await redis.set(`session:${sessionId}`, JSON.stringify(updatedData), 'KEEPTTL');
                
                return newTokens.access_token;
            })().finally(() => {
                refreshPromise = null; // Mở khóa khi xong việc
            });
        }

        try {
            // Tất cả request đến sau sẽ đứng chờ ở dòng này (await Promise)
            const newToken = await refreshPromise;
            
            // Retry request cũ với Token mới
            originalRequest.headers.Authorization = `Bearer ${newToken}`;
            return client(originalRequest);
        } catch (refreshErr) {
            console.error('❌ Refresh failed. Destroying session.');
            await redis.del(`session:${sessionId}`); // Logout user (Server-side)
            return Promise.reject(refreshErr);
        }
      }
      return Promise.reject(error);
    }
  );

  return client;
};
```

### 2. Catch-all Proxy Route (`frontend/app/api/proxy/[...path]/route.ts`)
Route này hứng mọi request từ Frontend gửi lên.

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import { createProxyClient } from '@/lib/proxy';

export async function GET(req: NextRequest, { params }: { params: { path: string[] } }) {
  // 1. Bóc tách SessionID từ Cookie
  const sessionId = cookies().get('session_id')?.value;
  if (!sessionId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  // 2. Xác định endpoint đích (VD: auth/me)
  const targetEndpoint = params.path.join('/');

  try {
    // 3. Khởi tạo Proxy Client (Đã bao gồm Smart Refresh)
    const client = createProxyClient(sessionId);
    
    // 4. Forward Request sang Laravel
    // QUAN TRỌNG: Forward IP thật của User để Laravel rate-limit đúng
    const { data } = await client.get(`/${targetEndpoint}`, {
        headers: {
            'X-Forwarded-For': req.headers.get('x-forwarded-for') || req.ip
        }
    });

    return NextResponse.json(data);
  } catch (e: any) {
    // Nếu BFF bó tay (Refresh thất bại) -> Trả 401 để Client logout
    return NextResponse.json({ error: e.message }, { status: e.response?.status || 500 });
  }
}
// (Bạn có thể thêm POST, PUT, DELETE tương tự tại đây)
```

### 3. Bảo mật thêm: CSRF Protection
Vì chúng ta dùng Cookie (`SameSite=Lax`), rủi ro CSRF vẫn hiện hữu (dù thấp).
Để an toàn tuyệt đối, hãy thêm cơ chế **Double Submit Cookie** cho các method biến đổi dữ liệu (POST/PUT/DELETE).

**Logic:**
1.  BFF tạo 1 token ngẫu nhiên `csrf_token` khi login.
2.  Gửi nó về Client qua Header hoặc Body (không phải cookie).
3.  Client khi gọi API Proxy phải gửi kèm Header `X-CSRF-Token`.
4.  BFF verify Header này khớp với Session trong Redis.

---

## VIII. Core Feature 4: Logout (Dọn dẹp sạch sẽ)

Đừng quên tính năng quan trọng nhất để bảo vệ user.

**File:** `frontend/app/api/auth/logout/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import redis from '@/lib/redis';

export async function POST(req: NextRequest) {
    const sessionId = cookies().get('session_id')?.value;

    if (sessionId) {
        // 1. Xóa trong Redis (Kill Session ngay lập tức)
        await redis.del(`session:${sessionId}`);
        
        // (Optional) Gọi Laravel để Blacklist Token hiện tại nếu cần
    }

    const response = NextResponse.json({ success: true });

    // 2. Xóa Cookie ở Browser
    response.cookies.delete('session_id');

    return response;
}
```

---

## IX. Hướng dẫn Test & Vận hành

Bây giờ hãy trải nghiệm sức mạnh của Automation.

### Bước 1: Khởi chạy ("One-Click")
Tại thư mục gốc `ultimate-auth`, mở terminal và chạy đúng 1 lệnh:

```bash
docker-compose up
```
Hãy đợi Docker tự động cài đặt Composer, NPM, tạo Database SQLite, sinh Key...

### Bước 2: Kiểm chứng Bảo mật
1.  Truy cập `http://localhost:3000`.
2.  Thực hiện Login.
3.  **DevTools -> Cookies:** Chỉ thấy `session_id`. Không có Token.
4.  **Local Storage:** Trống trơn.
5.  **Kết luận:** Front-end hoàn toàn "mù" về Token. Hacker XSS vô vọng.

### Bước 3: Kiểm chứng Smart Refresh
1.  Vào container backend, chỉnh file `config/jwt.php`: set `ttl = 1` (phút). Restart container backend.
2.  Login lại. Chờ 61 giây.
3.  Bấm F5 hoặc gọi API liên tục.
4.  **Quan sát Log Frontend:**
    Thấy dòng: `🔄 [BFF] Token expired. Calling Laravel Refresh....`
5.  Request vẫn trả về 200 OK.
6.  **Kết luận:** BFF đã tự động cứu request, User không hề biết token đã đổi.

---

## X. Lời kết Series

Hành trình 5 bài viết của chúng ta đã kết thúc. Chúng ta đã đi từ những khái niệm cơ bản đến việc tự tay xây dựng một hệ thống BFF Zero-Token hoàn chỉnh.

*   **Next.js** là "Vệ sĩ" tận tụy.
*   **Redis** là "Két sắt" an toàn.
*   **Laravel** là "Bộ não" thông minh.
*   **Docker** là công cụ giúp ta triển khai mọi thứ trong chớp mắt.

Dù kiến trúc này phức tạp hơn cách làm truyền thống, nhưng nó mang lại sự an tâm tuyệt đối cho các hệ thống Enterprise. Hy vọng series này đã trang bị đủ vũ khí để bạn tự tin chinh phục mọi bài toán Authentication.

Happy Coding & Stay Secure! 🚀

