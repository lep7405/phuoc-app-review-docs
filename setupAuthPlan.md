# Shopify Embedded App — Authentication Setup

## Tổng quan

App này là **Shopify Embedded App**, cần 2 flow authentication riêng biệt:

| Flow | Khi nào | Mục đích |
|---|---|---|
| **A. Install Flow** | Merchant cài app lần đầu | Lấy offline access token, lưu DB |
| **B. Request Flow** | Mỗi API request từ frontend | Verify session token, đổi lấy online token |

---

## Cấu trúc file

```
app/
├── Http/
│   ├── Controllers/
│   │   └── ShopifyController.php        ← Flow A: install + callback
│   ├── Middleware/
│   │   └── VerifyShopifySession.php     ← Flow B: verify JWT session token
├── Services/
│   └── ShopifyTokenExchange.php         ← Flow B: token exchange
├── Models/
│   └── ShopifyShop.php                  ← Eloquent model
database/
└── migrations/
    └── xxxx_create_shopify_shops_table.php
routes/
├── web.php                              ← /install, /callback
└── api.php                              ← protected API routes
.env                                     ← Shopify credentials
```

---

## Biến môi trường cần thiết

Thêm vào file `.env`:

```env
SHOPIFY_API_KEY=your_api_key_here
SHOPIFY_API_SECRET=your_api_secret_here
SHOPIFY_SCOPES=read_products,write_products
SHOPIFY_REDIRECT_URI=https://your-app.com/auth/callback
```

---

## FLOW A — Install Flow (Authorization Code Grant)

> Chạy **1 lần duy nhất** khi merchant cài app. Kết quả: lưu `offline access_token` vào DB.

### Sơ đồ

```
Merchant click Install trên Shopify App Store
  │
  ▼
GET /auth/install?shop=xxx.myshopify.com&hmac=...&timestamp=...
  │
  ├─ [A1] Validate shop domain (regex)
  ├─ [A2] Verify HMAC
  ├─ [A3] Tạo state/nonce → lưu session
  │
  ▼
Redirect → https://{shop}/admin/oauth/authorize?...
  │
  │ (Merchant bấm "Install" trên trang Shopify)
  │
  ▼
GET /auth/callback?code=...&hmac=...&state=...&shop=...&timestamp=...
  │
  ├─ [A4] Validate shop domain
  ├─ [A5] Verify state/nonce (chống CSRF)
  ├─ [A6] Verify HMAC
  ├─ [A7] POST đổi code → access_token
  ├─ [A8] Verify scopes
  ├─ [A9] Lưu vào DB (shopify_shops)
  │
  ▼
Redirect → App UI (https://{host}/apps/{api_key}/)
```

---

### A1. Validate shop domain

Mọi request có param `shop` đều phải validate trước.

**Regex:**
```
/^[a-zA-Z0-9][a-zA-Z0-9\-]*\.myshopify\.com$/
```

**Lý do:** Tránh attacker truyền domain giả để trick app gọi API tới server của họ.

---

### A2. Verify HMAC (install request)

Shopify ký mọi request gửi đến app bằng HMAC-SHA256. Phải verify trước khi xử lý.

**Thuật toán:**
1. Lấy tất cả query params từ request
2. Bỏ param `hmac` ra khỏi danh sách
3. Sort các params còn lại theo thứ tự alphabet (theo tên key)
4. Join thành query string dạng `key=value&key=value`
5. Tính `HMAC-SHA256(SHOPIFY_API_SECRET, query_string)`
6. So sánh kết quả (hex string) với giá trị `hmac` trong request

**Quan trọng:** Dùng **constant-time comparison** (`hash_equals()` trong PHP) để tránh timing attack.

```php
$params = $request->except('hmac');
ksort($params);
$queryString = http_build_query($params);
$computed = hash_hmac('sha256', $queryString, env('SHOPIFY_API_SECRET'));

if (!hash_equals($computed, $request->query('hmac'))) {
    abort(403, 'Invalid HMAC');
}
```

---

### A3. Tạo State / Nonce (chống CSRF)

Trước khi redirect sang Shopify, tạo một chuỗi random và lưu vào session.

```php
$state = bin2hex(random_bytes(16));
session(['shopify_oauth_state' => $state]);
```

Thêm `state` vào URL redirect:
```
https://{shop}/admin/oauth/authorize?...&state={state}
```

---

### A4. URL Redirect sang Shopify OAuth

```
https://{shop}/admin/oauth/authorize
  ?client_id={SHOPIFY_API_KEY}
  &scope={SHOPIFY_SCOPES}
  &redirect_uri={SHOPIFY_REDIRECT_URI}
  &state={state}
```

**Lưu ý `grant_options[]`:**
- Không truyền → **offline token** (không hết hạn, dùng cho background jobs)
- Truyền `grant_options[]=per-user` → **online token** (hết hạn theo session user)

Với embedded app, offline token vẫn cần cho background. Online token sẽ lấy qua Token Exchange ở Flow B.

---

### A5. Verify State / Nonce (callback)

Khi Shopify redirect về `/callback`, kiểm tra `state` param có khớp với giá trị đã lưu trong session không.

```php
if ($request->query('state') !== session('shopify_oauth_state')) {
    abort(403, 'State mismatch - possible CSRF attack');
}
session()->forget('shopify_oauth_state');
```

---

### A6. Verify HMAC (callback)

Tương tự A2, nhưng dùng toàn bộ params của callback request (bao gồm `code`, `host`, `state`, `timestamp`).

---

### A7. Đổi Code lấy Access Token

```http
POST https://{shop}/admin/oauth/access_token
Content-Type: application/x-www-form-urlencoded

client_id=SHOPIFY_API_KEY
client_secret=SHOPIFY_API_SECRET
code=authorization_code_from_callback
```

**Response:**
```json
{
  "access_token": "shpat_xxxxxxxxxxxx",
  "scope": "read_products,write_products"
}
```

Sử dụng `Http::post()` của Laravel:
```php
$response = Http::post("https://{$shop}/admin/oauth/access_token", [
    'client_id'     => env('SHOPIFY_API_KEY'),
    'client_secret' => env('SHOPIFY_API_SECRET'),
    'code'          => $request->query('code'),
]);

if ($response->failed()) {
    abort(500, 'Failed to get access token from Shopify');
}
```

---

### A8. Verify Scopes

Shopify có thể cấp ít scope hơn app yêu cầu (nếu merchant từ chối một số quyền). Cần kiểm tra.

```php
$grantedScopes = explode(',', $response->json('scope'));
$requiredScopes = explode(',', env('SHOPIFY_SCOPES'));

$missingScopes = array_diff($requiredScopes, $grantedScopes);

if (!empty($missingScopes)) {
    abort(403, 'Missing required scopes: ' . implode(',', $missingScopes));
}
```

---

### A9. Lưu vào Database

```php
ShopifyShop::updateOrCreate(
    ['shop_domain' => $shop],
    ['access_token' => $accessToken]
);
```

---

### A10. Redirect vào App UI

```php
$host = $request->query('host'); // base64 encoded host
return redirect("https://" . base64_decode($host) . "/apps/" . env('SHOPIFY_API_KEY') . "/");
```

---

## FLOW B — Request Flow (Session Token + Token Exchange)

> Chạy **mỗi request** từ frontend của embedded app. Kết quả: gọi Shopify API an toàn thay mặt merchant đang đăng nhập.

### Sơ đồ

```
Frontend (App Bridge) lấy session token (JWT, sống 1 phút)
  │
  ▼
Gửi request lên backend:
  Authorization: Bearer {session_token}
  │
  ▼
Middleware VerifyShopifySession:
  ├─ [B1] Tách JWT từ Authorization header
  ├─ [B2] Verify signature (HS256 + SHOPIFY_API_SECRET)
  ├─ [B3] Kiểm tra exp, nbf, iat
  ├─ [B4] Kiểm tra aud == SHOPIFY_API_KEY
  ├─ [B5] Kiểm tra dest là .myshopify.com hợp lệ
  │
  ▼
ShopifyTokenExchange::getOnlineToken($shop, $sessionToken):
  ├─ POST /admin/oauth/access_token
  │    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  │    subject_token={session_token}
  │    subject_token_type=urn:ietf:params:oauth:token-type:id_token
  │    requested_token_type=urn:ietf:params:oauth:token-type:access_token
  │
  ▼
Nhận online access_token
  │
  ▼
Gọi Shopify GraphQL API:
  X-Shopify-Access-Token: {online_access_token}
  │
  ▼
Trả response về frontend
```

---

### B1. Session Token là gì?

Session token là JWT do Shopify App Bridge tạo ra phía frontend. Token này:
- Sống **1 phút** (phải fetch mới mỗi request)
- Ký bằng **HS256** với key là `SHOPIFY_API_SECRET`
- Chứa thông tin về shop và user hiện tại

**JWT Payload:**
```json
{
  "iss": "https://example.myshopify.com/admin",
  "dest": "https://example.myshopify.com",
  "aud": "SHOPIFY_API_KEY",
  "sub": "user_id",
  "exp": 1234567890,
  "nbf": 1234567830,
  "iat": 1234567830,
  "jti": "uuid-random",
  "sid": "session-id"
}
```

---

### B2. Verify Session Token (Backend Middleware)

**Package cần cài:**
```bash
composer require firebase/php-jwt
```

**Checklist verify:**

| Claim | Điều kiện kiểm tra |
|---|---|
| Signature | Verify bằng `SHOPIFY_API_SECRET`, algo `HS256` |
| `exp` | `exp > time()` — token chưa hết hạn |
| `nbf` | `nbf <= time()` — token đã active |
| `aud` | Bằng `SHOPIFY_API_KEY` |
| `dest` | Khớp regex `.myshopify.com` |

Nếu bất kỳ check nào fail → trả về `401 Unauthorized`.

---

### B3. Token Exchange

Sau khi verify session token, backend dùng nó để đổi lấy **online access token**:

```http
POST https://{shop}/admin/oauth/access_token
Content-Type: application/x-www-form-urlencoded

client_id=SHOPIFY_API_KEY
client_secret=SHOPIFY_API_SECRET
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token={session_token}
subject_token_type=urn:ietf:params:oauth:token-type:id_token
requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

**Response:**
```json
{
  "access_token": "shpat_online_xxx",
  "scope": "read_products",
  "expires_in": 86399
}
```

**Online token này KHÔNG lưu DB** — chỉ dùng trong request hiện tại rồi bỏ.

---

### B4. Gọi Shopify API

```http
POST https://{shop}/admin/api/2025-01/graphql.json
X-Shopify-Access-Token: shpat_online_xxx
Content-Type: application/json

{ "query": "{ shop { name } }" }
```

---

## Database Schema

### Bảng `shopify_shops`

```php
Schema::create('shopify_shops', function (Blueprint $table) {
    $table->id();
    $table->string('shop_domain')->unique();   // example.myshopify.com
    $table->text('access_token');              // offline token, encrypt nếu cần
    $table->timestamps();
});
```

---

## Routes

```php
// routes/web.php — Flow A
Route::get('/auth/install',  [ShopifyController::class, 'install'])->name('shopify.install');
Route::get('/auth/callback', [ShopifyController::class, 'callback'])->name('shopify.callback');

// routes/api.php — Flow B (protected)
Route::middleware('shopify.session')->group(function () {
    Route::get('/api/shop', [ShopifyController::class, 'shop']);
    // thêm các route khác ở đây
});
```

---

## Checklist triển khai

### Flow A
- [ ] Tạo migration `shopify_shops`
- [ ] Tạo model `ShopifyShop`
- [ ] Tạo `ShopifyController` với method `install()` và `callback()`
- [ ] Thêm helper `verifyHmac()` và `validateShopDomain()`
- [ ] Đăng ký routes trong `web.php`
- [ ] Thêm biến môi trường vào `.env`
- [ ] Test: truy cập `/auth/install?shop=xxx.myshopify.com`

### Flow B
- [ ] Cài package `firebase/php-jwt`
- [ ] Tạo middleware `VerifyShopifySession`
- [ ] Tạo service `ShopifyTokenExchange`
- [ ] Đăng ký middleware trong `bootstrap/app.php`
- [ ] Bảo vệ API routes bằng middleware
- [ ] Test: gọi API với session token hợp lệ từ App Bridge

---

## Lưu ý bảo mật

1. **Không log access token** — tránh lộ token trong log files
2. **Encrypt access token trong DB** — dùng Laravel `encrypted` cast trên model
3. **Luôn dùng `hash_equals()`** khi so sánh HMAC — tránh timing attack
4. **Validate shop domain trước mọi thứ** — đây là input từ bên ngoài
5. **Không lưu online token** — chỉ dùng trong scope của 1 request
6. **SHOPIFY_API_SECRET phải giữ bí mật** — không commit lên git, không expose ra frontend
