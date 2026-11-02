# Chùa Online Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Xây dựng website Chùa Online (F1 đến F10 trong spec) bằng Node.js và MySQL, triển khai trên iNET Cloud VPS.

**Architecture:** Express phục vụ API và file tĩnh. Socket.IO xử lý realtime (online, chắp tay, tổng lạy). MySQL truy cập qua Knex. Trên VPS: Nginx làm reverse proxy có WebSocket, PM2 giữ app chạy, cron chạy `mysqldump` mỗi ngày.

**Tech Stack:** Node.js 20 LTS, Express, Socket.IO, mysql2, Knex, zod, helmet, express-rate-limit, express-session, bcryptjs, vitest, supertest, socket.io-client, ESLint, Prettier, Docker Compose (chỉ máy dev), PM2, Nginx, Certbot.

**Spec:** `docs/superpowers/specs/2026-11-02-chua-online-design.md`

## Global Constraints

- Node.js 20 LTS trở lên.
- MySQL 8.0.
- Múi giờ nghiệp vụ: Asia/Ho_Chi_Minh. Cột `day` được tính trong code, không dùng `CURDATE()`.
- Mỗi lần gửi lạy: số lạy từ 1 đến 200.
- Chắp tay: tối đa một lần mỗi 3 giây cho mỗi kết nối socket.
- `POST /api/bows`: tối đa 30 request mỗi phút cho mỗi IP.
- Không lưu danh tính người chắp tay.
- Cổng MySQL không mở ra internet (ufw chỉ mở 22, 80, 443).
- Biến môi trường bắt buộc: `PORT`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `ADMIN_PASSWORD_HASH`, `SESSION_SECRET`, `PUBLIC_ORIGIN`.
- Session ẩn danh: UUID v4, gửi qua header `X-Session-Id`.

## Review Focus

Năm trường hợp dễ gây lỗi nhất khi người dùng thật sử dụng. Mỗi dòng có test ở task sở hữu code.

1. **Lạy số lớn hoặc số âm/không hợp lệ** (ví dụ `count = -5`, `"abc"`, `1000000`): server trả 400, không ghi dữ liệu. *Test: Task 4, Task 6.*
2. **Hai tab cùng session gửi lạy cùng lúc:** kết quả là tổng cộng dồn, không mất lạy và không ghi đè. *Test: Task 4 (Promise.all).*
3. **Mất mạng giữa chừng hoặc tải lại trang:** số lạy chưa gửi được giữ trong localStorage và gửi lại khi có mạng. *Test: Task 7.*
4. **Chắp tay spam hoặc socket ngắt kết nối liên tục:** tần suất bị giới hạn, số online không âm và không bị tính sai khi kết nối lại. *Test: Task 9, Task 10.*
5. **Lạy sát nửa đêm (23:59 và 00:01 giờ Việt Nam):** lượt lạy được tách đúng ngày theo múi giờ Asia/Ho_Chi_Minh. *Test: Task 4.*

## Gate G0: Kiểm tra VPS trước khi viết code (không phải task code)

Thực hiện trên gói dùng thử 7 ngày của iNET. Không tiếp tục nếu có mục không đạt.

- [ ] Có quyền root và cài được Ubuntu LTS.
- [ ] Cài được Node 20, MySQL 8, Nginx, PM2, Certbot.
- [ ] Port 80 và 443 mở được từ bên ngoài.
- [ ] Một app Socket.IO đơn giản chạy qua Nginx với WebSocket hoạt động.
- [ ] Ghi lại toàn bộ lệnh đã dùng để đưa vào `deploy/setup-vps.md` (Task 13).

---

### Task 1: Khởi tạo dự án và Express skeleton

**Files:**
- Create: `package.json`, `.gitignore`, `.env.example`, `.eslintrc.cjs`, `.prettierrc`, `docker-compose.yml`
- Create: `server/config.js`, `server/app.js`, `server/index.js`
- Test: `tests/config.test.js`, `tests/app.test.js`

**Interfaces:**
- Produces: `loadConfig(env: NodeJS.ProcessEnv) -> Config` (zod, ném lỗi nếu thiếu biến bắt buộc). `createApp(deps: { config: Config, db?: Knex }) -> express.Application`.

- [ ] **Step 1: Khởi tạo git và npm**

Chạy `git init`, `npm init -y`, cài dependency chạy (`express`, `socket.io`, `zod`, `dotenv`) và dev (`vitest`, `supertest`, `socket.io-client`, `eslint`, `prettier`). Commit: `chore: init project`.

- [ ] **Step 2: Viết test cấu hình (thất bại)**

```js
// tests/config.test.js
test('loadConfig ném lỗi khi thiếu DB_PASSWORD', () => {
  expect(() => loadConfig({ ...validEnv, DB_PASSWORD: undefined })).toThrow(/DB_PASSWORD/);
});
test('loadConfig trả về PORT là số', () => {
  expect(loadConfig({ ...validEnv, PORT: '3000' }).port).toBe(3000);
});
```

Chạy `npx vitest run tests/config.test.js`. Kỳ vọng: FAIL vì chưa có module.

- [ ] **Step 3: Cài đặt `server/config.js`**

Dùng zod để parse đúng các biến trong Global Constraints. Trả về object có các trường camelCase (`port`, `db`, `adminPasswordHash`, `sessionSecret`, `publicOrigin`).

- [ ] **Step 4: Viết test app (thất bại)**

```js
test('GET /healthz trả 200 và { ok: true }', async () => {
  const res = await request(createApp({ config: testConfig })).get('/healthz');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ ok: true });
});
```

- [ ] **Step 5: Cài đặt `server/app.js` và `server/index.js`**

`createApp` đăng ký `express.json({ limit: '10kb' })`, route `/healthz`, và phục vụ `public/` làm static. `index.js` đọc config, tạo HTTP server và lắng nghe `config.port`.

- [ ] **Step 6: Chạy toàn bộ test để xác nhận**

Run: `npx vitest run`. Kỳ vọng: PASS cho cả hai file.

- [ ] **Step 7: Viết `docker-compose.yml` cho MySQL dev**

Service `mysql:8.0`, port `3306`, tạo sẵn database `chua_online` và `chua_online_test`. Chạy `docker compose up -d`, kỳ vọng `mysql -h127.0.0.1 -uroot -p` kết nối được.

- [ ] **Step 8: Commit**

`feat: express skeleton with config validation`

---

### Task 2: Kết nối MySQL, migration và seed

**Files:**
- Create: `knexfile.cjs`, `server/db/knex.js`, `server/db/migrations/001_init.js`, `server/db/seeds/001_sutras.js`
- Test: `tests/db.test.js`

**Interfaces:**
- Produces: `createKnex(dbConfig) -> Knex`. Migration tạo các bảng `sutras`, `visits`, `bows`, `daily_stats` theo spec mục 8.

- [ ] **Step 1: Viết test migration (thất bại)**

```js
test('migrate.latest tạo đủ 4 bảng với khóa chính đúng', async () => {
  await db.migrate.latest();
  expect(await db.schema.hasTable('bows')).toBe(true);
  // kiểm tra PRIMARY KEY của visits và bows là (session_id, day)
});
test('migrate.rollback xóa sạch các bảng', async () => { /* ... */ });
```

Chạy test, kỳ vọng FAIL.

- [ ] **Step 2: Cài đặt `server/db/knex.js` và migration**

Cột `session_id` là `CHAR(36)`, `day` là `DATE`, `count` và `bows_total` là `INT` và `BIGINT`. `sutras.is_active` là `BOOLEAN` mặc định `true`.

- [ ] **Step 3: Chạy migration trên database test và xác nhận test PASS**

Run: `NODE_ENV=test npx knex migrate:latest --knexfile knexfile.cjs && npx vitest run tests/db.test.js`. Kỳ vọng: PASS.

- [ ] **Step 4: Seed kinh mẫu**

Seed 3 bài kinh thuộc phạm vi công cộng, ghi rõ `source` và `license`. Ghi chú nguồn trong `README.md` (sẽ viết ở Task 13).

- [ ] **Step 5: Commit**

`feat: mysql connection, initial migration and sutra seed`

---

### Task 3: Repository sutras và `GET /api/sutras`

**Files:**
- Create: `server/repos/sutras.repo.js`, `server/routes/public.js`
- Modify: `server/app.js` (mount router)
- Test: `tests/sutras.test.js`

**Interfaces:**
- Produces: `listActiveSutras(db) -> Array<{ id, title, audioUrl, source }>` (sắp xếp theo `id`). `publicRouter({ db }) -> express.Router`.

- [ ] **Step 1: Viết test (thất bại)**

Test kiểm tra: chỉ trả về bài có `is_active = 1`; thứ tự theo `id`; response không chứa `license` hoặc `created_at`.

- [ ] **Step 2: Cài đặt repo và route**

Route `GET /api/sutras` trả `{ sutras: [...] }`.

- [ ] **Step 3: Chạy test, kỳ vọng PASS**

- [ ] **Step 4: Commit**

`feat: list active sutras endpoint`

---

### Task 4: Service ghi lạy và thống kê cá nhân

**Files:**
- Create: `server/services/time.js`, `server/services/bows.service.js`
- Test: `tests/bows.service.test.js`

**Interfaces:**
- Produces:
  - `dayKey(date: Date) -> string` (định dạng `YYYY-MM-DD`, theo Asia/Ho_Chi_Minh)
  - `recordBows(db, sessionId: string, count: number, now: Date) -> Promise<{ todayCount: number }>`
  - `getMyStats(db, sessionId: string, now: Date) -> Promise<{ todayCount: number, streakDays: number }>`
  - `class InvalidInputError extends Error`
- Ràng buộc: `count` là số nguyên từ 1 đến 200; `sessionId` đúng định dạng UUID v4; sai thì ném `InvalidInputError`.

- [ ] **Step 1: Viết test (thất bại)**

```js
test('cộng dồn hai lần: 10 rồi 5 cho ra 15', async () => { /* ... */ });
test('count 0, -5, 201, 1.5 đều ném InvalidInputError', async () => { /* ... */ });
test('23:59 và 00:01 giờ VN thuộc hai ngày khác nhau', () => {
  expect(dayKey(new Date('2026-11-02T16:59:00Z'))).toBe('2026-11-02'); // 23:59 VN
  expect(dayKey(new Date('2026-11-02T17:01:00Z'))).toBe('2026-11-03'); // 00:01 VN
});
test('20 lần recordBows(1) song song cho tổng 20', async () => {
  await Promise.all(Array.from({ length: 20 }, () => recordBows(db, sid, 1, now)));
  expect((await getMyStats(db, sid, now)).todayCount).toBe(20);
});
```

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt `time.js` và `bows.service.js`**

`recordBows` chạy trong transaction: upsert vào `bows` bằng `ON DUPLICATE KEY UPDATE count = count + VALUES(count)`, sau đó cộng vào `daily_stats.bows_total` cùng ngày. `getMyStats` tính streak là số ngày liên tiếp có `count > 0`, kể từ hôm nay hoặc hôm qua.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Commit**

`feat: bow recording with timezone-aware day keys`

---

### Task 5: Service lượt truy cập và thống kê hôm nay

**Files:**
- Create: `server/services/visits.service.js`
- Test: `tests/visits.service.test.js`

**Interfaces:**
- Produces:
  - `recordVisit(db, sessionId: string, now: Date) -> Promise<{ firstToday: boolean }>` (dùng `INSERT IGNORE`)
  - `getTodayStats(db, now: Date) -> Promise<{ visits: number, bowsTotal: number }>`

- [ ] **Step 1: Viết test (thất bại)**

Cùng một session gọi `recordVisit` hai lần trong ngày: lần đầu `firstToday: true`, lần hai `false`, `visits` tăng đúng 1. Sang ngày mới thì đếm lại.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt service**

`recordVisit` cộng `daily_stats.visits` khi `firstToday`. `getTodayStats` đọc từ `daily_stats` theo `dayKey(now)`, trả 0 nếu chưa có dòng.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Commit**

`feat: unique daily visits and today stats`

---

### Task 6: Endpoint ghi lạy, thống kê và rate limit

**Files:**
- Create: `server/middleware/rateLimit.js`, `server/middleware/errors.js`, `server/middleware/session.js`
- Modify: `server/routes/public.js`
- Test: `tests/api.bows.test.js`

**Interfaces:**
- Produces:
  - `POST /api/bows` với header `X-Session-Id` và body `{ count }`. Trả `200 { todayCount }`, `400` nếu không hợp lệ, `429` nếu vượt giới hạn.
  - `GET /api/stats/me` với header `X-Session-Id`. Trả `200 { todayCount, streakDays }`.
  - `GET /api/stats/today`. Trả `200 { visits, bowsTotal }`.
- Ràng buộc: `express-rate-limit` cho `POST /api/bows` là 30 request mỗi phút cho mỗi IP.

- [ ] **Step 1: Viết test (thất bại)**

```js
test('POST /api/bows count=0 trả 400', async () => { /* ... */ });
test('POST /api/bows count="abc" trả 400', async () => { /* ... */ });
test('POST /api/bows vượt 30 request/phút trả 429', async () => { /* gửi 31 request */ });
test('GET /api/stats/today trả số nguyên không âm', async () => { /* ... */ });
```

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt middleware và route**

`session.js` đọc `X-Session-Id`, từ chối nếu không phải UUID v4 (trả 400). `errors.js` chuyển `InvalidInputError` thành 400 và các lỗi khác thành 500 không lộ chi tiết.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Commit**

`feat: bow and stats endpoints with rate limiting`

---

### Task 7: Trang chính điện: lạy và đếm 108

**Files:**
- Create: `public/index.html`, `public/css/style.css`, `public/js/session.js`, `public/js/bow-batcher.js`, `public/js/main.js`
- Test: `tests/bow-batcher.test.js`

**Interfaces:**
- Produces:
  - `getSessionId() -> string` (localStorage, bọc try/catch, tạo UUID v4 nếu chưa có).
  - `createBowBatcher({ send: (count) => Promise<void>, storage, intervalMs = 5000, maxBatch = 200 }) -> { add(n), flush(), pending() }`

- [ ] **Step 1: Viết test batcher (thất bại)**

```js
test('add(3) rồi flush() gọi send(3) đúng một lần', async () => { /* ... */ });
test('send lỗi thì giữ lại pending và gửi lại lần flush sau', async () => { /* ... */ });
test('pending lưu trong storage, instance mới đọc lại được', () => { /* ... */ });
test('add(450) rồi flush() tách thành các lần gửi tối đa 200', async () => { /* ... */ });
```

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt `bow-batcher.js`**

Lưu `pending` vào `storage` (localStorage bọc try/catch) mỗi khi `add` hoặc `flush`. Khi `send` thất bại, không xóa `pending`.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Cài đặt giao diện chính điện**

Màn hình sảnh cho chọn không gian (chính điện, sân chùa) theo F1. Trong chính điện: nút lạy (108 lần), bộ đếm, hiệu ứng khi lạy, âm thanh mõ có nút bật/tắt. Gọi `flush()` mỗi 5 giây và khi `visibilitychange` sang `hidden`. Khi tải trang, gọi `GET /api/stats/me` để hiển thị số lạy hôm nay.

- [ ] **Step 6: Kiểm tra thủ công**

Mở trình duyệt, lạy 10 lần, tải lại trang, thấy số lạy được giữ. Tắt mạng rồi lạy, bật mạng lại, xác nhận lạy được gửi.

- [ ] **Step 7: Commit**

`feat: bow page with batched submission`

---

### Task 8: Tràng hạt, trình phát kinh, thắp hương

**Files:**
- Create: `public/js/rosary.js`, `public/js/player.js`, `public/js/incense.js`
- Test: `tests/rosary.test.js`, `tests/player.test.js`

**Interfaces:**
- Produces:
  - `renderRosary(el, { total = 108, onWrap })` trả về `{ tap(), count() }`. Đến hạt thứ 108 thì gọi `onWrap` và đưa về 0.
  - `nextSutra(list, currentId) -> Sutra | null` (thuần, không DOM).
  - `lightIncense(container, durationMs = 8000) -> () => void` (hàm dọn dẹp).

- [ ] **Step 1: Viết test (thất bại)**

```js
test('tap 108 lần gọi onWrap đúng một lần và count() về 0', () => { /* ... */ });
test('nextSutra trả bài kế tiếp, và quay lại bài đầu ở cuối danh sách', () => { /* ... */ });
```

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt ba module**

Tràng hạt dùng 108 phần tử, đánh dấu hạt đã lần bằng class CSS. Trình phát dùng thẻ `<audio>` với `src` là `audioUrl` của bài đã chọn. Thắp hương tạo phần tử khói bằng CSS animation và dọn sau `durationMs`.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Kiểm tra thủ công**

Lần đủ 108 hạt, nghe một bài kinh, thắp hương, xác nhận khói tự tắt.

- [ ] **Step 6: Commit**

`feat: rosary, sutra player and incense`

---

### Task 9: Realtime bằng Socket.IO

**Files:**
- Create: `server/sockets/index.js`, `server/sockets/presence.js`, `server/sockets/greet.js`
- Modify: `server/index.js` (gắn Socket.IO vào HTTP server)
- Test: `tests/sockets.test.js`, `tests/greet.limiter.test.js`

**Interfaces:**
- Produces:
  - `attachSockets(httpServer, { db }) -> Server`
  - `createGreetLimiter({ windowMs = 3000 }) -> (socketId: string, now: number) => boolean`
- Sự kiện:
  - Client gửi `join` với `{ sessionId }`.
  - Server phát `online:count` với `{ count }` khi có kết nối hoặc ngắt kết nối. Số đếm lấy từ `io.engine.clientsCount`.
  - Client gửi `greet`. Server phát `greeting:new` với `{ at }` cho tất cả client. Không kèm danh tính. Nếu vượt giới hạn thì bỏ qua lặng lẽ.
  - Server phát `stats:today` với `{ visits, bowsTotal }`, tối đa mỗi 5 giây.

- [ ] **Step 1: Viết test limiter (thất bại)**

Gọi `allow('a', 0)` trả `true`, `allow('a', 2999)` trả `false`, `allow('a', 3000)` trả `true`. Socket khác nhau có bộ đếm riêng.

- [ ] **Step 2: Viết test socket (thất bại)**

Hai client kết nối, kỳ vọng `online:count` là 2. Một client ngắt, kỳ vọng còn 1. Client A gửi `greet` hai lần liền, client B chỉ nhận đúng một `greeting:new`, và payload chỉ có key `at`.

- [ ] **Step 3: Chạy test, kỳ vọng FAIL**

- [ ] **Step 4: Cài đặt**

`greet.js` lưu thời điểm gửi cuối của từng socket trong Map và xóa khi ngắt kết nối. `presence.js` phát `online:count`. Khi `join`, server ghi nhận `visit` qua `recordVisit` từ Task 5.

- [ ] **Step 5: Chạy test, kỳ vọng PASS**

- [ ] **Step 6: Commit**

`feat: realtime presence, greetings and stats broadcast`

---

### Task 10: Client realtime

**Files:**
- Create: `public/js/realtime.js`
- Modify: `public/js/main.js`, `public/index.html`
- Test: `tests/realtime.test.js`

**Interfaces:**
- Produces: `connectRealtime({ url, sessionId, onOnline, onGreeting, onStats, socketFactory }) -> { greet(), disconnect() }`. `socketFactory` cho phép test thay bằng bản giả.

- [ ] **Step 1: Viết test (thất bại)**

Với socket giả: `greet()` hai lần liền chỉ emit một lần (cooldown 3 giây phía client). Khi kết nối lại (`reconnect`), `join` được gửi lại với cùng `sessionId`. `onOnline` nhận đúng số từ server.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt `realtime.js`, nối vào giao diện**

Hiển thị số online, thông báo chắp tay ngắn (tự ẩn sau 3 giây), và cập nhật tổng lạy từ `stats:today`.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Kiểm tra thủ công**

Mở hai trình duyệt. Chắp tay ở một bên, bên kia thấy thông báo ngay. Tắt mạng một bên rồi bật lại, số online đúng.

- [ ] **Step 6: Commit**

`feat: client realtime integration`

---

### Task 11: Trang quản trị

**Files:**
- Create: `server/services/auth.service.js`, `server/middleware/requireAdmin.js`, `server/routes/admin.js`, `public/admin/index.html`, `public/admin/admin.js`
- Modify: `server/repos/sutras.repo.js` (thêm `createSutra`, `updateSutra`, `deleteSutra`), `server/app.js` (gắn `express-session`)
- Test: `tests/admin.test.js`

**Interfaces:**
- Produces:
  - `verifyAdminPassword(plain: string, hash: string) -> Promise<boolean>`
  - `adminRouter({ db, config }) -> express.Router` gồm `POST /api/admin/login`, `POST /api/admin/logout`, `GET/POST/PUT/DELETE /api/admin/sutras`, `GET /api/admin/stats`.
- Cookie phiên: `httpOnly`, `sameSite: 'lax'`, `secure` khi `NODE_ENV=production`. Lưu ý: `express-session` mặc định dùng bộ nhớ, chỉ phù hợp một instance.

- [ ] **Step 1: Viết test (thất bại)**

- Đăng nhập sai mật khẩu trả 401.
- Đăng nhập đúng trả 200 và có cookie.
- `GET /api/admin/sutras` không có cookie trả 401.
- Tạo bài kinh qua POST thì xuất hiện trong danh sách.
- `GET /api/admin/stats` trả số không âm.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt**

Middleware `requireAdmin` kiểm tra `req.session.isAdmin`. Route CRUD dùng zod để validate `title`, `audioUrl`, `source`, `license`.

- [ ] **Step 4: Chạy test, kỳ vọng PASS**

- [ ] **Step 5: Kiểm tra thủ công**

Đăng nhập admin, thêm một bài kinh, mở trang khách và thấy bài đó.

- [ ] **Step 6: Commit**

`feat: admin login and sutra management`

---

### Task 12: Bảo mật và kiểm tra spam

**Files:**
- Modify: `server/app.js` (helmet, CORS giới hạn theo `PUBLIC_ORIGIN`)
- Create: `scripts/spam-check.js`
- Test: `tests/security.test.js`

**Interfaces:**
- Produces: `scripts/spam-check.js` gửi 100 request `POST /api/bows` và in số lượng `200` và `429`.

- [ ] **Step 1: Viết test (thất bại)**

- Response có header `X-Content-Type-Options: nosniff`.
- Request có `Origin` khác `PUBLIC_ORIGIN` không nhận header `Access-Control-Allow-Origin`.
- Body lớn hơn 10kb trả 413.
- `X-Session-Id` có ký tự SQL như `' OR 1=1 --` bị từ chối với 400.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL**

- [ ] **Step 3: Cài đặt helmet và CORS, viết `spam-check.js`**

- [ ] **Step 4: Chạy test, kỳ vọng PASS. Chạy `node scripts/spam-check.js` và xác nhận có 429 trong kết quả.**

- [ ] **Step 5: Commit**

`feat: security headers, CORS and spam check script`

---

### Task 13: Artifacts triển khai VPS và sao lưu

**Files:**
- Create: `deploy/nginx.conf`, `ecosystem.config.js`, `deploy/backup.sh`, `deploy/restore.sh`, `deploy/setup-vps.md`, `README.md`
- Test: thủ công trên máy dev, ghi kết quả vào `deploy/setup-vps.md`

**Interfaces:**
- `deploy/backup.sh`: chạy `mysqldump` vào thư mục backup, giữ 7 bản gần nhất. Nhận biến `DB_NAME`, `DB_USER`, `BACKUP_DIR`.
- `deploy/restore.sh <file>`: khôi phục vào database `chua_online_restore_check`.
- `deploy/nginx.conf`: proxy `/` và `/socket.io/` tới `127.0.0.1:3000`, có `Upgrade` và `Connection` cho WebSocket.

- [ ] **Step 1: Kiểm tra backup trên máy dev**

Chạy `deploy/backup.sh`, kỳ vọng tạo một file `.sql.gz`. Chạy lại 8 lần, kỳ vọng còn đúng 7 file.

- [ ] **Step 2: Kiểm tra khôi phục**

Chạy `deploy/restore.sh` với file mới nhất, so sánh số dòng `sutras` và `bows` với database gốc. Kỳ vọng khớp.

- [ ] **Step 3: Viết `ecosystem.config.js`, `nginx.conf`, `setup-vps.md`**

`setup-vps.md` là bản chép lại các lệnh từ Gate G0, theo đúng thứ tự.

- [ ] **Step 4: Viết README**

Gồm: cách chạy dev, biến môi trường, nguồn kinh và bản quyền, lệnh deploy.

- [ ] **Step 5: Commit**

`chore: deployment artifacts and backup scripts`

---

### Task 14: Triển khai production và nghiệm thu

**Files:**
- Create: `scripts/smoke-check.js`

**Interfaces:**
- `scripts/smoke-check.js <baseUrl>`: kiểm tra `GET /healthz`, `GET /api/sutras`, kết nối Socket.IO và nhận `online:count`. In PASS hoặc FAIL cho từng mục.

- [ ] **Step 1: Mua VPS và tên miền, trỏ DNS**

Kiểm tra `nslookup <domain>` trả về IP của VPS.

- [ ] **Step 2: Cài SSL và chạy app**

Chạy `certbot --nginx -d <domain>`. Chạy `pm2 start ecosystem.config.js`, rồi `pm2 save` và `pm2 startup`. Chạy migration production: `NODE_ENV=production npx knex migrate:latest`.

- [ ] **Step 3: Chạy smoke check**

Run: `node scripts/smoke-check.js https://<domain>`. Kỳ vọng: tất cả mục PASS.

- [ ] **Step 4: Kiểm tra thủ công trên điện thoại**

Mở link, lạy, lần tràng hạt, nghe kinh, chắp tay. Kiểm tra cron backup đã chạy: `ls` thư mục backup có file mới.

- [ ] **Step 5: Đối chiếu tiêu chí nghiệm thu trong spec mục 10**

Ghi kết quả từng mục vào README.

- [ ] **Step 6: Commit**

`chore: production deployment and acceptance notes`

---

## Thứ tự và phụ thuộc

Gate G0 → Task 1 → Task 2 → Task 3 → Task 4 → Task 5 → Task 6 → Task 7 → Task 8 → Task 9 → Task 10 → Task 11 → Task 12 → Task 13 → Task 14.

Task 7 và Task 8 chỉ phụ thuộc Task 6 (API). Task 8 có thể làm song song với Task 9 nếu có hai người.
