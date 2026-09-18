# Nx Module Federation — điều tra bug share scope

> Bản Markdown/ASCII của `nx-module-federation.drawio` (5 page). Nội dung chữ, đoạn code và
> tham chiếu file/số dòng được giữ đúng như trong diagram gốc.

**Hệ thống:** monorepo Nx + Angular (share scope trong diagram ghi phiên bản `19.2.20`).

| Vai trò | App | URL |
| --- | --- | --- |
| HOST | `gridsz-client` | `http://localhost:4200` |
| REMOTE | `gridsz-module-ac` | `http://localhost:4201` |

**Triệu chứng:** build qua Docker thì Angular libs bị phục vụ từ REMOTE (4201) thay vì HOST (4200),
dẫn tới `NullInjectorError`. Chạy `nx serve` thì không lỗi.

**Kết luận:** đặt `shareStrategy: 'loaded-first'` (mục 4).

## Quy ước đọc ASCII art

| Ký hiệu | Ý nghĩa (màu trong diagram gốc) |
| --- | --- |
| `[HOST]` | HOST `gridsz-client` @ 4200 — xanh dương `#dae8fc` |
| `[REMOTE]` | REMOTE `gridsz-module-ac` @ 4201 — xanh lá `#d5e8d4` |
| `✓` | luồng đúng / kết quả mong muốn — xanh lá |
| `✗` `[LỖI]` | luồng lỗi / triệu chứng của bug — đỏ `#f8cecc` |
| `[!]` | build tooling, dữ liệu runtime, ghi chú quan trọng — vàng `#fff2cc` |
| `[CFG]` | file config / artifact — xám `#f5f5f5` |
| khung nét đôi `╔═╗` | container / nhóm lớn trong page gốc |
| khung nét đơn `┌─┐` | một box trong page gốc |

Mỗi mục có ghi chú legend riêng khi màu ở page đó mang nghĩa khác.

## Mục lục

1. [Bối cảnh & triệu chứng](#1-bối-cảnh--triệu-chứng)
2. [Nguyên nhân gốc: race condition của shareStrategy = version-first](#2-nguyên-nhân-gốc-race-condition-của-sharestrategy--version-first)
3. [Vì sao nx serve chạy được mà Docker/prod thì không](#3-vì-sao-nx-serve-chạy-được-mà-dockerprod-thì-không)
4. [Các phương án đã thử](#4-các-phương-án-đã-thử)
5. [Kiến trúc Module Federation (build-time + runtime)](#5-kiến-trúc-module-federation-build-time--runtime)

---

## 1. Bối cảnh & triệu chứng

Page gốc: **1. Setup & Symptoms**. Monorepo Nx có một host `gridsz-client`
(http://localhost:4200) và một remote `gridsz-module-ac` (http://localhost:4201) nối với
nhau bằng Module Federation, cùng share một tập thư viện Angular. Hai cách chạy dùng đúng
cùng một `module-federation.config.ts`, nhưng khi `nx serve` thì HOST phục vụ các Angular
libs, còn khi build qua Docker thì REMOTE phục vụ chúng — và một khi remote thắng
`shareScope`, hậu quả cuối cùng là `NullInjectorError` tại runtime.

Legend dùng cho toàn mục: `[HOST]` = gridsz-client @ 4200 · `[REMOTE]` = gridsz-module-ac
@ 4201 · `[CFG]` = file config/artifact · `[!]` = ghi chú quan trọng / dữ liệu runtime ·
`✓` = hành vi đúng · `✗` = hành vi lỗi.

### 1.1 Kiến trúc

```text
╔═ Architecture ═══════════════════════════════════════════════════════════════════════════╗
║          ┌───────────────────────┐                                                       ║
║          │ [!] Browser           │                                                       ║
║          └───────────────────────┘                                                       ║
║                      │                                                                   ║
║                      ▼                                                                   ║
║ ┌────────────────────────────────────────┐   ┌────────────────────────────────────────┐  ║
║ │ [HOST] gridsz-client                   │   │ [REMOTE] gridsz-module-ac              │  ║
║ │ http://localhost:4200                  │   │ http://localhost:4201                  │  ║
║ ├────────────────────────────────────────┤   ├────────────────────────────────────────┤  ║
║ │ module-federation.config.ts            │   │ module-federation.config.ts            │  ║
║ │ • name: 'gridsz-client'                │   │ • exposes: 5 module/component          │  ║
║ │ • remotes: []  (đăng ký tại runtime)   │   │   (xem fence bên dưới)                 │  ║
║ │ • shared: @angular/*, rxjs, zone.js,   │   │ • shared: SAME set as host             │  ║
║ │   @ngrx/*, @angular/cdk, material,     │   └────────────────────────────────────────┘  ║
║ │   moment, leaflet, ...                 │                                               ║
║ └────────────────────────────────────────┘                                               ║
║                                                                                          ║
║ ┌─────────────────────────────────────────────────────────────────────────────────────┐  ║
║ │ [CFG] apps/gridsz-client/src/assets/module-federation.manifest.json                 │  ║
║ │ { "gridsz-module-ac": "http://localhost:4201/mf-manifest.json" }                    │  ║
║ └─────────────────────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
```

**`module-federation.config.ts` — HOST `gridsz-client`** (trích nguyên văn từ diagram):

```text
• name: 'gridsz-client'
• remotes: [] (registered at runtime)
• shared: @angular/*, rxjs, zone.js,
  @ngrx/*, @angular/cdk, material, moment, leaflet, ...
```

**`module-federation.config.ts` — REMOTE `gridsz-module-ac`** (trích nguyên văn):

```text
• exposes:
    './AcModule','./ADMModule','./ACEngineBoostModule',
    './AddressConnectionComponent',
    './DialogCreatePolygonComponent'
• shared: SAME set as host
```

**`apps/gridsz-client/src/assets/module-federation.manifest.json`**:

```json
{ "gridsz-module-ac": "http://localhost:4201/mf-manifest.json" }
```

### 1.2 Triệu chứng quan sát trên DevTools Network tab

```text
╔═ Triệu chứng quan sát trên DevTools Network tab ═════════════════════════════════════════╗
║ ┌────────────────────────────────────┐                                                   ║
║ │ nx serve (dev server)              │  ⇒  ✓ Angular libs do HOST (4200) phục vụ         ║
║ │ pnpm serve:client + pnpm serve:ac  │                                                   ║
║ └────────────────────────────────────┘                                                   ║
║                                                                                          ║
║ ┌────────────────────────────────────┐                                                   ║
║ │ Docker (Dockerfile.dev)            │  ⇒  ✗ Angular libs do REMOTE (4201) phục vụ       ║
║ │ build:development → nginx static   │       (build executor không có Nx runtime plugin) ║
║ └────────────────────────────────────┘                                                   ║
║                                                                                          ║
║ ┌──────────────────────────────────────────────────────────────────────────────────────┐ ║
║ │ [!] Câu hỏi cốt lõi                                                                  │ ║
║ │ Cả 2 cấu hình đều dùng cùng module-federation.config.ts.                             │ ║
║ │ Vì sao chỉ nx serve thắng scope (host phục vụ libs),                                 │ ║
║ │ còn build qua Docker đều thua (remote phục vụ libs)?                                 │ ║
║ └──────────────────────────────────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
```

| Cách chạy | Lệnh / cấu hình | Ai phục vụ Angular libs? |
|---|---|---|
| `nx serve` (dev server) | `pnpm serve:client` + `pnpm serve:ac` | ✓ HOST (4200) |
| Docker (`Dockerfile.dev`) | `build:development` → nginx static | ✗ REMOTE (4201) `*` |

`*` Lý do ghi trong diagram: build executor không có Nx runtime plugin.

### 1.3 Hậu quả khi remote thắng shareScope

```text
╔═ Hậu quả khi remote thắng shareScope ════════════════════════════════════════════════════╗
║   ┌──────────────────────────────────────────────────────────────────────────────────┐   ║
║   │ ① Remote tải sớm                                                                 │   ║
║   │ remoteEntry.mjs (localhost:4201) được fetch ngay trong lúc host.init() chạy      │   ║
║   │ (version-first auto-load mọi remote)                                             │   ║
║   └──────────────────────────────────────────────────────────────────────────────────┘   ║
║                                             │                                            ║
║                                             ▼                                            ║
║   ┌──────────────────────────────────────────────────────────────────────────────────┐   ║
║   │ ② Remote đăng ký dependency vào scope                                            │   ║
║   │ container.init(scope) register @angular/*, rxjs, ngrx, cdk, material...          │   ║
║   │ với from='gridsz-module-ac', loaded:true                                         │   ║
║   │ TRƯỚC khi host kịp đăng ký bản của mình                                          │   ║
║   └──────────────────────────────────────────────────────────────────────────────────┘   ║
║                                             │                                            ║
║                                             ▼                                            ║
║   ┌──────────────────────────────────────────────────────────────────────────────────┐   ║
║   │ ③ Host KHÔNG tải lib từ domain của nó                                            │   ║
║   │ Khi host import @angular/core, MF resolver tra shareScope → thấy entry của       │   ║
║   │ remote đã loaded → trả về copy của remote.                                       │   ║
║   │ Chunk @angular/core của host (4200) không bao giờ được fetch hay evaluate.       │   ║
║   └──────────────────────────────────────────────────────────────────────────────────┘   ║
║                                             │                                            ║
║                                             ▼                                            ║
║   ┌──────────────────────────────────────────────────────────────────────────────────┐   ║
║   │ ④ NullInjectorError tại runtime                                                  │   ║
║   │ Provider của host (bootstrapApplication, providers: [...]) được declare          │   ║
║   │ bằng symbol của host's Angular bundle.                                           │   ║
║   │ inject(...) tra qua injector của remote's Angular instance → không khớp          │   ║
║   │ symbol → NullInjectorError: No provider for X                                    │   ║
║   └──────────────────────────────────────────────────────────────────────────────────┘   ║
║                                                                                          ║
║   ┌──────────────────────────────────────────────────────────────────────────────────┐   ║
║   │ [!] Vì sao có NullInjectorError?  → diễn giải đầy đủ ở bullet ngay dưới khung    │   ║
║   └──────────────────────────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
```

**[!] Vì sao có NullInjectorError?**

Cả host và remote đều bundle riêng code Angular của mình (mỗi bundle có
class/symbol/decorator metadata được tree-shake riêng). MF chỉ khóa **MỘT** instance
Angular tại runtime — nếu remote thắng scope:

- Mọi `import @angular/core` trong host code đều resolve về phiên bản **RUNTIME** của remote.
- Nhưng các `provider`/`InjectionToken` được biên dịch trong host bundle vẫn dùng symbol
  identity riêng của host's compile-time Angular.
- Khi component host gọi `inject(TOKEN)`, framework dùng injector tree của remote's Angular
  instance. Token instance trong injector đó (nếu có) lại là của remote, **KHÔNG bằng (`===`)**
  `TOKEN` của host code.
- Lookup fail → `NullInjectorError: No provider for ServiceX!`

---

## 2. Nguyên nhân gốc: race condition của shareStrategy = version-first

*(page gốc: `2. Root cause: race condition with shareStrategy = 'version-first' (default)`)*

Mục này dựng lại timeline của host `gridsz-client` từ lúc `main.js` được evaluate cho tới
lúc share scope bị remote chiếm, đặt cạnh những gì quan sát được trên Network tab và đoạn
code runtime tương ứng. Điểm cốt lõi: với `shareStrategy = 'version-first'` (giá trị mặc
định), `host.init()` tự động init **mọi** remote, nên `remoteEntry.mjs` của `4201` được nạp
trước khi host kịp load chunk Angular của chính nó — và `register()` cho phép entry đến sau
ghi đè entry đến trước.

Legend: `[HOST]` = host `gridsz-client` @ 4200 (xanh dương) · `[LỖI]` = bước/triệu chứng lỗi
(đỏ) · `[!]` = ghi chú quan trọng (vàng) · `[nhánh giả định]` = box nét đứt ở source, tức
tình huống *nếu* dùng `loaded-first`/`hostWins`, **không** phải quan sát thực tế của bug.

### 2.1 Timeline của HOST đối chiếu Network tab

Cột trái là chuỗi thời gian `t1 → t2 → t3 → t4 → t5 → t6` (đúng theo các mũi tên của page
gốc). Cột phải là những gì xuất hiện trên Network tab, xếp ngang hàng với bước tương ứng theo
đúng toạ độ của page gốc. Page gốc còn một cột thứ ba — `Code reference
(runtime-core/shared/index.js)` — rộng tới mức không thể đặt cạnh hai cột kia trong 100 cột,
nên nó được tách xuống §2.2; hai marker `[xem code ①]` / `[xem code ②]` trong art là phần
thêm vào để giữ lại liên kết đó, page gốc không có.

```text
HOST timeline (browser executes main.js)              What appears in Network tab

┌──────────────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ [HOST] t1: main.js evaluate webpack auto-init    │  │ [HOST] localhost:4200/main.js            │
│        host MF container                         │  │        localhost:4200/vendor.js          │
│ (name='gridsz_client', shared declared,          │  │        localhost:4200/polyfills.js       │
│ but chunks not loaded yet)                       │  │        localhost:4200/bootstrap.ts.js    │
└──────────────────────────────────────────────────┘  └──────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ [HOST] t2: main.ts user code runs                │  │ [nhánh giả định — nét đứt ở source]      │
│ fetch('module-federation.manifest.json')         │  │ (if loaded-first or hostWins lock)       │
│ registerPlugins([fallbackPlugin])                │  │ Host shared chunks load next             │
│ registerRemotes([{name:'gridsz-module-ac',...}]) │  │ and populate scope first                 │
└──────────────────────────────────────────────────┘  └──────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ [HOST] t3: import('./bootstrap')                 │  │ [LỖI] (version-first, default)           │
│ bootstrap.ts loads (async)                       │  │ localhost:4201/remoteEntry.mjs ←         │
│ triggers Angular imports →                       │  │ then 70+ chunks from 4201:               │
│ __webpack_require__('@angular/core')             │  │   angular_core, angular_common,          │
│                                                  │  │   angular_cdk, angular_material,         │
│                                                  │  │   angular_router, rxjs, ...              │
│                                                  │  │ Only ~12 chunks from 4200.               │
└──────────────────────────────────────────────────┘  └──────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────┐
│ [LỖI] t4: Inside host.init() —                   │
│       runtime-core/shared/index.js               │
│ register host's @angular/core in scope           │
│ THEN (version-first only):                       │
│   remotes.forEach(initRemoteModule)              │
│   ← auto-loads remoteEntry.mjs of EVERY remote!  │
│                                    [xem code ①]  │
└──────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────┐
│ [LỖI] t5: remoteEntry.mjs loaded from 4201       │
│ runtime calls container.init(scope)              │
│ remote registers its @angular/core version       │
│ into the same scope                              │
└──────────────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────────────┐
│ [LỖI] t6: Race resolved by register() logic      │
│ If host chunk not yet loaded:                    │
│   remote's entry has loaded:true first           │
│   → wins singleton resolution                    │
│ App now uses REMOTE's Angular instance           │
│                                    [xem code ②]  │
└──────────────────────────────────────────────────┘
```

Đọc nhanh từng bước:

| Bước | Điều xảy ra ở host | Dấu hiệu trên Network tab |
| --- | --- | --- |
| `t1` | `main.js` được evaluate, webpack auto-init MF container của host (`name='gridsz_client'`, phần `shared` đã khai báo, nhưng chunk thì chưa nạp) | 4 file của 4200: `main.js`, `vendor.js`, `polyfills.js`, `bootstrap.ts.js` |
| `t2` | Code người dùng trong `main.ts` chạy: `fetch('module-federation.manifest.json')`, `registerPlugins([fallbackPlugin])`, `registerRemotes([{name:'gridsz-module-ac',...}])` | *(nhánh giả định)* nếu có `loaded-first` hoặc lock `hostWins` thì shared chunk của host nạp kế tiếp và điền vào scope trước |
| `t3` | `import('./bootstrap')` → `bootstrap.ts` load async → kéo theo các import Angular → `__webpack_require__('@angular/core')` | **`version-first` (mặc định)**: `localhost:4201/remoteEntry.mjs`, rồi 70+ chunk từ 4201 (`angular_core`, `angular_common`, `angular_cdk`, `angular_material`, `angular_router`, `rxjs`, …); chỉ ~12 chunk đến từ 4200 |
| `t4` | Trong `host.init()` (`runtime-core/shared/index.js`): đăng ký `@angular/core` của host vào scope, **rồi** (chỉ với `version-first`) `remotes.forEach(initRemoteModule)` → auto-load `remoteEntry.mjs` của MỌI remote | — |
| `t5` | `remoteEntry.mjs` từ 4201 đã nạp, runtime gọi `container.init(scope)`, remote đăng ký version `@angular/core` của nó vào **cùng** scope đó | — |
| `t6` | Race được phân xử bởi logic `register()`: nếu chunk của host chưa nạp xong thì entry của remote có `loaded:true` trước → thắng singleton resolution. App giờ dùng instance Angular của REMOTE | — |

### 2.2 Tham chiếu code (`runtime-core/shared/index.js`)

① Đoạn khiến remote bị auto-init — `runtime-core/shared/index.js`,
trong `host.init()`, ~line 192:

```js
// inside host.init() — line ~192
if (host.options.shareStrategy === 'version-first'
    || strategy === 'version-first') {
  host.options.remotes.forEach((remote) => {
    if (remote.shareScope === shareScopeName)
      promises.push(initRemoteModule(remote.name));
    // ← this is what triggers remote auto-init
  });
}
```

② Đoạn quyết định ai thắng share scope — `runtime-core/shared/index.js`,
trong `register()`, ~line 164:

```js
// inside register() — line ~164
if (!activeVersion
    || activeVersion.strategy !== 'loaded-first'
       && !activeVersion.loaded
       && (eager-or-hostName-comparison))
  versions[version] = shared;  // overwrite entry
// With 'loaded-first', existing entries are STICKY
// — no overwrite, even if a remote tries later.
```

### 2.3 Kết luận

```text
╔════════════════════════════════════════════════════════════════════════════════════════════════╗
║ [!] Kết luận                                                                                   ║
║ Mặc định 'version-first' có HAI hành vi gây ra bug:                                            ║
║ 1. Tự động load mọi remote ngay trong host.init() → race bắt đầu lập tức.                      ║
║ 2. Cho phép các lần register sau ghi đè entry đã có trong scope, dựa trên                      ║
║    eager/version/hostName.                                                                     ║
║ Kết quả: ở prod build (không eager, không Nx runtime plugin), remote có thể                    ║
║ thắng và chiếm share scope.                                                                    ║
╚════════════════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 3. Vì sao nx serve chạy được mà Docker/prod thì không

*(page gốc: `3. Why nx serve works (and Docker prod doesn't)`)*

Cùng một cấu hình Module Federation, nhưng `nx serve` đi qua executor
`@nx/angular:module-federation-dev-server` còn Docker/prod đi qua `@nx/angular:webpack-browser`.
Chỉ nhánh dev-server mới set biến môi trường `NX_MF_DEV_REMOTES`, và chính biến đó là điều kiện
để Nx bơm thêm một runtime plugin ép host thắng mọi lần resolve package shared. Nhánh Docker không
có biến đó nên bundle ra không có plugin, và resolver mặc định của MF để remote thắng scope race.

Legend: `✓` = box xanh lá (nhánh đúng / OK) · `[LỖI]` = box đỏ (bước hoặc triệu chứng lỗi) ·
`[!]` = box vàng (ghi chú quan trọng) · box không nhãn = box xanh dương, ở page này là bước
trung gian của pipeline (xanh dương **không** mang nghĩa `[HOST]` như các mục khác) ·
`①②③④` = bốn tầng của luồng · `▼` = đúng 6 mũi tên (edge) của diagram gốc.

### 3.1 So sánh song song hai nhánh, 4 tầng

Hai hộp tiêu đề chỉ là nhãn của cột (source không có mũi tên từ tiêu đề xuống tầng ①). Hộp
`Takeaway` nằm full-width bên dưới cả hai cột, đúng như vị trí của nó trong page gốc.

```text
╔══════════════════════════════════════════════╗   ╔══════════════════════════════════════════════╗
║              nx serve (works ✓)              ║   ║       Docker (any build:development /        ║
║                                              ║   ║         build:production) — broken ✗         ║
╚══════════════════════════════════════════════╝   ╚══════════════════════════════════════════════╝

┌──────────────────────────────────────────────┐   ┌──────────────────────────────────────────────┐
│ ① Executor:                                  │   │ ① Executor:                                  │
│   @nx/angular:module-federation-dev-server   │   │   @nx/angular:webpack-browser                │
│   (xem target "serve" trong project.json)    │   │   (gọi từ Dockerfile.dev hoặc pnpm build)    │
└──────────────────────────────────────────────┘   └──────────────────────────────────────────────┘
                        │                                                  │
                        ▼                                                  ▼
┌──────────────────────────────────────────────┐   ┌──────────────────────────────────────────────┐
│ ② start-remote-iterators.js, line 22-25      │   │ ② [LỖI] Không có                             │
│ đặt process.env.NX_MF_DEV_REMOTES =          │   │   module-federation-dev-server trong chain   │
│   JSON.stringify([...devRemoteNames, HOST])  │   │ ⇒ process.env.NX_MF_DEV_REMOTES là           │
│ → xem code (a)                               │   │   undefined                                  │
└──────────────────────────────────────────────┘   └──────────────────────────────────────────────┘
                        │                                                  │
                        ▼                                                  ▼
┌──────────────────────────────────────────────┐   ┌──────────────────────────────────────────────┐
│ ③ with-module-federation.js,                 │   │ ③ [LỖI] with-module-federation.js thấy       │
│   line 59-65 & 72-74                         │   │   env var = undefined                        │
│ có env ⇒ runtimePlugins =                    │   │ ⇒ ModuleFederationPlugin.runtimePlugins      │
│   [ runtime-library-control.plugin.js ]      │   │   = undefined                                │
│ + DefinePlugin định nghĩa                    │   │ ⇒ không có DefinePlugin entry cho            │
│   process.env.NX_MF_DEV_REMOTES              │   │   NX_MF_DEV_REMOTES                          │
│ → xem code (b)                               │   │ ⇒ bundle thiếu library-control plugin        │
└──────────────────────────────────────────────┘   └──────────────────────────────────────────────┘
                        │                                                  │
                        ▼                                                  ▼
┌──────────────────────────────────────────────┐   ┌──────────────────────────────────────────────┐
│ ④ ✓ runtime-library-control.plugin.js        │   │ ④ [LỖI] Khi chạy, resolver mặc định của      │
│   (at runtime): resolveShare()               │   │   MF chạy:                                   │
│ HOST đứng đầu __INSTANCES__ (auto-init)      │   │   findSingletonVersionOrderByVersion()       │
│ VÀ nằm trong devRemotes                      │   │   + register() với logic version-first       │
│ ⇒ host thắng mọi shared resolve  ✓           │   │   + host.init() tự trigger                   │
│ → xem code (c)                               │   │     initRemoteModule cho mọi remote          │
│                                              │   │ ⇒ remote thường thắng scope race             │
│                                              │   │ ⇒ Angular chunks tải từ localhost:4201 ✗     │
└──────────────────────────────────────────────┘   └──────────────────────────────────────────────┘

╔═════════════════════════════════════════════════════════════════════════════════════════════════╗
║ [!] Takeaway                                                                                    ║
║                                                                                                 ║
║ Nx che mất vấn đề ở môi trường dev bằng cách bơm một runtime plugin gắn với một biến            ║
║ môi trường. Plugin đó CHỈ dành cho dev — production không bao giờ nhận được nó. Nên             ║
║ cùng một cấu hình MF lại hành xử khác nhau giữa dev và prod.                                    ║
║                                                                                                 ║
║ Mọi cách sửa an toàn cho production đều KHÔNG được dựa vào runtime plugin dev-only              ║
║ của Nx.                                                                                         ║
╚═════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### 3.2 Code mà nhánh `nx serve` chạy qua

**(a)** `start-remote-iterators.js`, line 22-25 — tầng ② của nhánh `nx serve`:

```js
process.env.NX_MF_DEV_REMOTES = JSON.stringify([
  ...devRemoteNames.map(normalize),
  normalize(project.name),     // ← HOST included in list
]);
```

**(b)** `with-module-federation.js`, line 59-65 & 72-74 — tầng ③ của nhánh `nx serve`:

```js
if (process.env.NX_MF_DEV_REMOTES) {
  mfPluginOptions.runtimePlugins = [
    require.resolve('runtime-library-control.plugin.js'),
  ];
}
new DefinePlugin({
  'process.env.NX_MF_DEV_REMOTES': process.env.NX_MF_DEV_REMOTES,
});
```

**(c)** `runtime-library-control.plugin.js` (at runtime) — tầng ④ của nhánh `nx serve`:

```js
resolveShare(args) {
  args.resolver = function () {
    const dev = GlobalFederation.__INSTANCES__.find(i =>
      i.options.shared[pkgName]
      && runtimeStore.devRemotes.includes(i.name));
    if (!dev) return originalResolver();
    return dev.options.shared[pkgName].find(s => s.from === dev.name);
  };
}
```

Vì HOST đứng đầu `__INSTANCES__` (được auto-init) VÀ tên host cũng nằm trong `devRemotes`, nhánh
`if` này luôn tìm thấy `dev` là chính host → host thắng mọi lần resolve package shared.

### 3.3 Nhánh Docker — chuỗi nguyên nhân

| Tầng | Trạng thái ở nhánh Docker (`build:development` hoặc `build:production`) |
|---|---|
| ① | Executor là `@nx/angular:webpack-browser`, được gọi từ `Dockerfile.dev` hoặc `pnpm build` |
| ② | Không có `module-federation-dev-server` trong chain → `process.env.NX_MF_DEV_REMOTES` là `undefined` |
| ③ | `with-module-federation.js` thấy env var `undefined` → `ModuleFederationPlugin.runtimePlugins = undefined`, không có DefinePlugin entry cho `NX_MF_DEV_REMOTES`, bundle xuất ra thiếu library-control plugin |
| ④ | Runtime dùng resolver mặc định của MF: `findSingletonVersionOrderByVersion()`, `register()` với logic version-first, cộng thêm `host.init()` tự trigger `initRemoteModule` cho mọi remote → remote thường thắng scope race → Angular chunks bị fetch từ `localhost:4201` |

---

## 4. Các phương án đã thử

Mục này đặt cạnh nhau 4 hướng xử lý việc remote ghi đè shared scope của host (A, B, C, D) và chốt
phương án **D — `shareStrategy: 'loaded-first'`**: dùng đúng setting chính thức của Module Federation,
không phình bundle, không phải viết code riêng. Phần sau ghi lại thay đổi đã áp dụng, hai tác động
trong `runtime-core/shared/index.js` kèm số dòng nguồn, và trình tự runtime cuối cùng.

### 4.1 Bốn phương án song song

```text
┌───────────────────────┬───────────────────────┬─────────────────────────┬───────────────────────┐
│ A. eager: true        │ B. Custom             │ C. setRemoteDefinitions │ D. shareStrategy:     │
│    on shared deps     │    hostWinsPlugin     │    + loadRemoteModule   │    'loaded-first'     │
│                       │                       │                         │                       │
│ [!] đã thử            │ [!] đã thử            │ [!] đã thử              │ ✓ ADOPTED (ĐƯỢC CHỌN) │
├───────────────────────┼───────────────────────┼─────────────────────────┼───────────────────────┤
│ Cơ chế                │ Cơ chế                │ Cơ chế (API Nx cũ)      │ Cơ chế (setting MF)   │
│ gộp shared của host   │ plugin MF tự viết,    │ init sharing tường minh │ 2 tác động trong      │
│ vào initial chunk →   │ hook resolveShare →   │ trước mỗi               │ runtime-core/         │
│ register đồng bộ      │ trả entry của host    │ container.init          │ shared/index.js       │
│ trước remote init     │ (đăng ký đồng bộ)     │                         │ → xem mục 4.4         │
├───────────────────────┼───────────────────────┼─────────────────────────┼───────────────────────┤
│ Chi phí               │ Chi phí               │ Chi phí                 │ Chi phí               │
│ +300-500 kB gzip      │ HOST_NAME hardcode    │ @nx/angular/mf          │ 0 kB, 0 dòng code,    │
│ (Angular core+cdk+    │ → lệch khi rename;    │ deprecated (mất ở       │ API chính thức        │
│ material+rxjs)        │ code tự bảo trì       │ Nx 22)                  │                       │
├───────────────────────┼───────────────────────┼─────────────────────────┼───────────────────────┤
│ Kết luận              │ Kết luận              │ Kết luận                │ Kết luận              │
│ ✓ được, nhưng đổi     │ ✓ được, nhưng làm     │ ✓ cùng ý tưởng,         │ ✓ host register       │
│   size lấy an toàn    │   lại việc MF có sẵn  │ ✗ API đã deprecated     │   trước, remote không │
│                       │                       │                         │   ghi đè được về sau  │
└───────────────────────┴───────────────────────┴─────────────────────────┴───────────────────────┘
                                                                                      │
                                                                                      ▼
╔═════════════════════════════════════════════════════════════════════════════════════════════════╗
║ ADOPTED: shareStrategy: 'loaded-first'   (áp dụng cho host gridsz-client @ 4200)                ║
╚═════════════════════════════════════════════════════════════════════════════════════════════════╝
```

Legend: `[!]` = phương án đã thử nhưng không chọn (ô vàng trong diagram gốc) · `✓` = phương án được
chọn / kết quả đúng (ô xanh lá) · `✗` = nhược điểm chặn.

| Phương án | Cơ chế | Chi phí | Kết luận |
| --- | --- | --- | --- |
| **A. `eager: true` on shared deps** | Gộp các shared module của host vào initial chunk. Chúng register đồng bộ **trước** khi bất kỳ `container.init` của remote nào chạy. | + Bundle ban đầu ~+300-500 kB gzip (Angular core+cdk+material+rxjs) | Chạy được, nhưng đánh đổi bundle size để lấy an toàn. |
| **B. Custom `hostWinsPlugin`** | Plugin MF runtime tự viết, hook `resolveShare`. Với mọi shared package, trả về entry của host lấy từ `__INSTANCES__`. Phải được register **ĐỒNG BỘ** ở đầu `main.ts`. | + `HOST_NAME` hardcode — rủi ro lệch khi rename.<br>+ Code tự viết phải bảo trì. | Chạy được, nhưng phát minh lại thứ MF đã hỗ trợ sẵn. |
| **C. `setRemoteDefinitions` + `loadRemoteModule`** (API Nx cũ) | `loadRemoteContainer()` gọi tường minh `await __webpack_init_sharing__('default')` trước mỗi `container.init`. Việc đó nạp sẵn shares của host vào scope một cách đồng bộ. | + `@nx/angular/mf` đã deprecated (mất hẳn ở Nx 22). | Cùng ý tưởng với `loaded-first`, nhưng đứng trên API đã deprecated. |
| **D. `shareStrategy: 'loaded-first'` ✓ ADOPTED (ĐƯỢC CHỌN)** (setting chính thức của MF) | Hai tác động trong `runtime-core/shared/index.js`: **Effect 1 (line ~192)** và **Effect 2 (line ~164)** — nội dung đầy đủ ở mục 4.4. | + Không có: 0 kB thêm vào bundle, 0 dòng code tự viết, API chính thức. | **Kết quả:** host register trước, remote không thể ghi đè về sau. |

### 4.2 Trước / sau khi đổi sang `loaded-first`

Hai tác động ở mục 4.4 làm thay đổi trình tự nạp như sau — nhánh trái là hành vi mặc định
`'version-first'` đã gây lỗi, nhánh phải là hành vi sau khi đặt `'loaded-first'`:

```text
┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
│ TRƯỚC — 'version-first' (mặc định)   │   │ SAU — 'loaded-first' ✓               │
├──────────────────────────────────────┤   ├──────────────────────────────────────┤
│                                      │   │                                      │
│ host.init()                          │   │ host.init()                          │
│   └─ index.js ~192 CHẠY:             │   │   └─ index.js ~192 BỊ SKIP:          │
│      auto-init tất cả remotes        │   │      không auto-init remote nào      │
│         ▼                            │   │         ▼                            │
│ remote container.init() chạy sớm,    │   │ host chunks register @angular/* vào  │
│ index.js ~164: register() ghi đè     │   │ share scope TRƯỚC (sticky entry)     │
│ entry @angular/* trong share scope   │   │         ▼                            │
│         ▼                            │   │ loadRemote(...) → remote init;       │
│ ✗ App dùng Angular instance của      │   │ index.js ~164: register() bị         │
│   REMOTE → remote thắng              │   │ short-circuit, không ghi đè          │
│   singleton resolution               │   │         ▼                            │
│                                      │   │ ✓ Angular của HOST vẫn là            │
│                                      │   │   singleton duy nhất                 │
└──────────────────────────────────────┘   └──────────────────────────────────────┘
```

### 4.3 Thay đổi đã áp dụng

Thay đổi nằm ở `apps/gridsz-client/webpack.config.ts`, truyền qua `configOverride` vào
`withModuleFederation`:

```ts
// apps/gridsz-client/module-federation.config.ts (or via configOverride)
const config: ModuleFederationConfig = {
  name: 'gridsz-client',
  remotes: [],
  shared: (libraryName, defaultConfig) => {
    if (librariesNotToShare.has(libraryName)) return false;
    return defaultConfig;
  },
};
export default config;

// apps/gridsz-client/webpack.config.ts
const wmf = await withModuleFederation(config, { dts: false, shareStrategy: 'loaded-first' });
return wmf({ ...wco, plugins: [..., new MomentLocalesPlugin(...)] });
```

### 4.4 Hai tác động của `loaded-first` (kèm số dòng nguồn)

**Effect 1 — Lazy remote init** — `runtime-core/shared/index.js, ~line 192`

- Bỏ qua bước auto-load-remotes trong `host.init()`.
- Remote chỉ nạp khi code người dùng gọi `loadRemote(...)`.

```js
if (host.options.shareStrategy === 'version-first' || strategy === 'version-first') {
  host.options.remotes.forEach((remote) => { promises.push(initRemoteModule(...)); });
}
// 'loaded-first' → this block is SKIPPED
// Remote loads only when user calls loadRemote(name/Module).
```

**Effect 2 — Sticky scope entries** — `runtime-core/shared/index.js, ~line 164 (inside register())`

- `register()` từ chối ghi đè một entry đã có trong scope nếu strategy của nó là `'loaded-first'`.
- Entry trở thành *sticky*.

```js
if (!activeVersion
    || activeVersion.strategy !== 'loaded-first'
       && !activeVersion.loaded
       && (eager-or-hostname-comparison)) versions[version] = shared;
// 'loaded-first' on the existing entry → SHORT CIRCUIT, no overwrite.
```

### 4.5 Hành vi runtime cuối cùng

```text
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║ Hành vi runtime cuối cùng                                                                ║
║ [HOST] gridsz-client @4200  +  [REMOTE] gridsz-module-ac @4201                           ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ① main.js chạy → webpack tự init MF instance của host.                                   │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ② main.ts gọi registerRemotes (chỉ URL, chưa fetch gì).                                  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ③ import('./bootstrap') nạp các shared chunk của host → host register @angular/* vào     │
│    scope.                                                                                │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ④ Không remote nào tự nạp (vì 'loaded-first').                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ⑤ Khi user điều hướng vào route federated, loadRemote('gridsz-module-ac/AcModule') fetch │
│    remoteEntry.mjs.                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ⑥ container.init của remote chạy. register() thấy entry 'loaded-first' đang có → KHÔNG   │
│    ghi đè. Angular của host vẫn là singleton.                                            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

Kiểm chứng trong DevTools console:

```js
window.__FEDERATION__.__SHARE_SCOPE_MAP__.default['@angular/core']  // entry.from === 'gridsz_client'
```

---

## 5. Kiến trúc Module Federation (build-time + runtime)

Mục này ghép hai nửa của cùng một hệ thống: lúc **build**, webpack + `ModuleFederationPlugin`
biến source của HOST và REMOTE thành hai bộ artifact độc lập (mỗi bên có `remoteEntry.mjs` và
`mf-manifest.json` riêng); lúc **runtime** trong browser, host chạy chuỗi ① → ⑤ để tự khởi tạo
MF instance, nạp danh sách remote từ manifest, boot Angular, rồi mới `loadRemote` module của
remote khi người dùng vào `/ac`. `module-federation.config.ts` là nguồn sự thật cho nửa build,
còn `__SHARE_SCOPE_MAP__` là nơi hai bên gặp nhau ở nửa runtime — chính chỗ sinh ra bug.

### 5.1 BUILD TIME — hai nhánh webpack song song

Trong diagram gốc hai nhánh này nằm ngang (source → webpack → output) và xếp thành 2 hàng
(HOST ở trên, REMOTE ở dưới); ở đây mỗi nhánh được trải theo chiều dọc cho dễ đọc.

```text
╔════════════════════════════════════════════════════════════════════════════════════════════════╗
║ BUILD TIME                                                                                     ║
║                                                                                                ║
║ Nhánh HOST (gridsz-client @ 4200):                                                             ║
║  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ HOST source                                                                       [HOST] │  ║
║  │ apps/gridsz-client/                                                                      │  ║
║  │ • src/main.ts                                                                            │  ║
║  │ • src/bootstrap.ts                                                                       │  ║
║  │ • module-federation.config.ts                                                            │  ║
║  │ • webpack.config.ts                                                                      │  ║
║  │ • module-federation.manifest.json                                                        │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ webpack                                                                              [!] │  ║
║  │ + ModuleFederationPlugin                                                                 │  ║
║  │ (@module-federation/enhanced/webpack)                                                    │  ║
║  │ + withModuleFederation (Nx wrapper)                                                      │  ║
║  │                                                                                          │  ║
║  │ Reads MF config → injects:                                                               │  ║
║  │ • name, exposes, remotes, shared                                                         │  ║
║  │ • library: { type: 'module' }                                                            │  ║
║  │ • runtimePlugins (dev only)                                                              │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ HOST output (dist/apps/gridsz-client)                                             [HOST] │  ║
║  │ • main.js (entry, MF auto-init code)                                                     │  ║
║  │ • vendor.js, polyfills.js                                                                │  ║
║  │ • remoteEntry.mjs (host as container)                                                    │  ║
║  │ • mf-manifest.json (declares shares/exposes)                                             │  ║
║  │ • async chunks for shared deps:                                                          │  ║
║  │     angular_core_..., angular_common_...,                                                │  ║
║  │     rxjs_..., etc.                                                                       │  ║
║  │ • assets/module-federation.manifest.json                                                 │  ║
║  │     (maps remote name → entry URL)                                                       │  ║
║  └──────────────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                                ║
║ Nhánh REMOTE (gridsz-module-ac @ 4201):                                                        ║
║  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ REMOTE source                                                                   [REMOTE] │  ║
║  │ apps/gridsz-module-ac/                                                                   │  ║
║  │ • module-federation.config.ts                                                            │  ║
║  │     exposes: { './AcModule': ..., }                                                      │  ║
║  │ • webpack.config.ts                                                                      │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ webpack                                                                              [!] │  ║
║  │ + ModuleFederationPlugin                                                                 │  ║
║  │ + withModuleFederation                                                                   │  ║
║  │                                                                                          │  ║
║  │ Generates a container with:                                                              │  ║
║  │ • exposes (the public API)                                                               │  ║
║  │ • shared (must overlap with host)                                                        │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ REMOTE output (dist/apps/gridsz-module-ac)                                      [REMOTE] │  ║
║  │ • remoteEntry.mjs (container with init/get)                                              │  ║
║  │ • mf-manifest.json                                                                       │  ║
║  │ • exposed module chunks (AcModule, ...)                                                  │  ║
║  │ • shared dep chunks (Angular, rxjs, ...)                                                 │  ║
║  │ Served at http://localhost:4201/                                                         │  ║
║  └──────────────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                                ║
║ Đầu vào chung của cả hai nhánh:                                                                ║
║  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ module-federation.config.ts (the source of truth)                                  [CFG] │  ║
║  │ Cả hai bước webpack ở trên đều đọc file này (xem mục 5.2 để lấy nội dung).               │  ║
║  └──────────────────────────────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════════════════════════════╝
```

Điểm cần chú ý: HOST cũng là một container (`remoteEntry.mjs` được sinh cho cả host), và
`assets/module-federation.manifest.json` của HOST mới là chỗ map `tên remote → entry URL` —
`remotes: []` trong config HOST là cố ý, vì remote được đăng ký ở runtime.

### 5.2 `module-federation.config.ts` — "the source of truth"

Panel này nằm bên phải khung BUILD TIME trong diagram gốc, cao ngang cả hai hàng và không có
mũi tên nào nối vào — nó là dữ liệu đầu vào mà bước webpack của mỗi nhánh đọc.

HOST — nguồn: `apps/gridsz-client/module-federation.config.ts`

```ts
{
  name: 'gridsz-client',
  remotes: [],            // resolved at runtime via registerRemotes
  shared: (lib) => ({ singleton, strictVersion, ... }),
}
```

REMOTE — nguồn: `apps/gridsz-module-ac/module-federation.config.ts`

```ts
{
  name: 'gridsz-module-ac',
  exposes: {
    './AcModule':
      'apps/gridsz-module-ac/src/app/ac-remote-entry/entry.routes.ts',
    './AddressConnectionComponent': ...,
  },
  shared: (lib) => ({ singleton, strictVersion, ... }),
}
```

### 5.3 RUNTIME (in the browser) — chuỗi ① → ⑤

Khung RUNTIME trong diagram gốc là khung nét đứt, gồm chuỗi 5 bước có mũi tên
(Browser → ① → ② → ③ → ④ → ⑤) cộng panel tham chiếu ⑥ không nối vào luồng.

```text
╔════════════════════════════════════════════════════════════════════════════════════════════════╗
║ RUNTIME (in the browser)                                                                       ║
║                                                                                                ║
║  ┌────────────────────┐                                                                        ║
║  │ Browser            │                                                                        ║
║  └─────────┬──────────┘                                                                        ║
║            │                                                                                   ║
║            ▼                                                                                   ║
║  ┌─────────┴────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ ① Load main.js + polyfills.js + vendor.js                                         [HOST] │  ║
║  │ • webpack runtime auto-inits HOST MF instance:                                           │  ║
║  │   - sets options: name, shared declarations,                                             │  ║
║  │     remotes (empty for this app), runtimePlugins                                         │  ║
║  │   - registers HOST in GlobalFederation.__INSTANCES__                                     │  ║
║  │ • Exposes globals:                                                                       │  ║
║  │   - __webpack_share_scopes__                                                             │  ║
║  │   - __FEDERATION__ (.__SHARE_SCOPE_MAP__, __INSTANCES__)                                 │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ ② main.ts (user code) executes                                                    [HOST] │  ║
║  │ registerPlugins → fetch manifest → registerRemotes → import('./bootstrap')               │  ║
║  │ mã nguồn đầy đủ: mục 5.4                                                                 │  ║
║  │                                                                                          │  ║
║  │ Note: registerRemotes only stores URLs.                                                  │  ║
║  │ No remote fetch happens here.                                                            │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ ③ bootstrap.ts evaluates                                                          [HOST] │  ║
║  │ bootstrapApplication(AppComponent, appConfig)                                            │  ║
║  │ • imports @angular/core, common, router, ...                                             │  ║
║  │ • webpack runtime resolves each via shared scope                                         │  ║
║  │ • populates scope with HOST's versions                                                   │  ║
║  │   (entry.from = 'gridsz_client')                                                         │  ║
║  │ • Angular boots → router activates initial route                                         │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ ④ User navigates to /ac → router needs AcModule                                 [REMOTE] │  ║
║  │ loadRemote('gridsz-module-ac/AcModule')                                                  │  ║
║  │                                                                                          │  ║
║  │ The MF runtime:                                                                          │  ║
║  │ • Looks up remote URL from registerRemotes                                               │  ║
║  │ • Fetches http://localhost:4201/remoteEntry.mjs                                          │  ║
║  │ • Calls container.init(__webpack_share_scopes__.default)                                 │  ║
║  │   → remote registers ITS shared versions in scope                                        │  ║
║  │   (singleton check decides who wins)                                                     │  ║
║  │ • Calls container.get('./AcModule')                                                      │  ║
║  │ • Returns factory → router consumes                                                      │  ║
║  └──────────────────────┬───────────────────────────────────────────────────────────────────┘  ║
║                         │                                                                      ║
║                         ▼                                                                      ║
║  ┌──────────────────────┴───────────────────────────────────────────────────────────────────┐  ║
║  │ ⑤ __FEDERATION__.__SHARE_SCOPE_MAP__.default                                         [!] │  ║
║  │ nội dung map: mục 5.4                                                                    │  ║
║  │                                                                                          │  ║
║  │ Lookup rule (singleton):                                                                 │  ║
║  │ • version-first → highest version, first-registered wins ties                            │  ║
║  │ • loaded-first → existing scope entry is STICKY, no overwrite                            │  ║
║  └──────────────────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                                ║
║  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ ⑥ Public Runtime API  -  @module-federation/enhanced/runtime                    [REMOTE] │  ║
║  │ panel tham chiếu: mọi lời gọi ở ②/④ đều đến từ API này (danh sách đầy đủ ở 5.5)          │  ║
║  └──────────────────────────────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### 5.4 Code chi tiết của ② và ⑤

② `main.ts` — nguồn: `apps/gridsz-client/src/main.ts`

```ts
registerPlugins([fallbackPlugin(), ...])

fetch('/assets/module-federation.manifest.json')
  .then(parse)
  .then(remotes => registerRemotes(remotes))
  .then(() => import('./bootstrap'))
```

⑤ Dữ liệu runtime trong browser, tại `__FEDERATION__.__SHARE_SCOPE_MAP__.default`

```js
{
  '@angular/core': {
    '19.2.20': { from: 'gridsz_client', loaded: true,
                 strategy: 'loaded-first', get: () => ... }
  },
  '@angular/common': { ... },
  'rxjs': { ... },
  ...
}
```

### 5.5 ⑥ Public Runtime API — `@module-federation/enhanced/runtime`

| Nhóm | Thành phần |
|---|---|
| Hàm chính | `registerRemotes([{ name, entry }])` • `loadRemote('name/Module') ⇒ Promise` • `registerPlugins([plugin])` • `preloadRemote([...])` • `getInstance() ⇒ ModuleFederation \| null` |
| Shared | `loadShare` • `loadShareSync` • `registerShared` |
| Lifecycle hooks (plugins) | `beforeRegisterRemote` • `afterRegisterRemote` • `beforeRequest` • `afterLoadShare` • `resolveShare` • `errorLoadRemote` • ... |

### 5.6 Legend (nguyên văn hộp `Legend` của page gốc)

Quy ước dùng cho cả tài liệu nằm ở đầu file ([Quy ước đọc ASCII art](#quy-ước-đọc-ascii-art)).
Dưới đây là hộp `Legend` của page 5 trong diagram gốc, màu được thay bằng nhãn text:

| Màu trong drawio | Nhãn ASCII | Ý nghĩa |
|---|---|---|
| Xanh dương | `[HOST]` | HOST (gridsz-client @ 4200) — giữ shell, routes, và là Angular runtime đang hoạt động khi nó thắng scope. |
| Xanh lá | `[REMOTE]` / `✓` | REMOTE (gridsz-module-ac @ 4201) — các module được expose + một container cùng tham gia scope. |
| Vàng | `[!]` | Build tooling / dữ liệu runtime dùng chung. |
| Xám | `[CFG]` | Artifact / file config. |
| Đỏ | `✗` / `[LỖI]` | Lỗi hoặc triệu chứng xấu. |
