
# APISIX
## HLD
---
### 1. Các thành phần của APISIX

- Control Plane = nơi khai báo, lưu trữ ý định — "tôi muốn route này chạy thế nào, plugin nào, upstream nào"
- Data Plane = nơi thực thi — nhận request thực, chạy plugin chain, forward xuống RGW
- Các đối tượng như Route, Upstream, Consumer... là definition/config → Control Plane. Data Plane chỉ consume config đó để xử lý traffic. Khi etcd die, Data Plane vẫn tiếp tục chạy bình thường với config đã load vào memory — đây là lý do APISIX tách bạch rõ 2 plane.

```
# Bảng tóm tắt
DATA PLANE                                CONTROL PLANE
───────────────────────────────────       ──────────────────────────────
APISIX Core (OpenResty/Nginx + Lua)       Dashboard (optional)
Router Engine (radixtree)                 Admin API (:9180)             
Plugin Runtime (Lua chain)                etcd (config store)                
Balancer (chọn upstream node)               ├─ Route
TLS Engine (dynamic cert)                   ├─ Upstream
Connection Pool → RGW backend               ├─ Service
                                            ├─ Plugin config
                                            ├─ Consumer
                                            ├─ SSL
                                            └─ Global Rule
```

#### Data Plane (xử lý traffic thực tế)
| Thành phần         | Vai trò                                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------------------------- |
| APISIX Core        | Engine chính, built on OpenResty/Nginx + Lua. Xử lý toàn bộ request pipeline congdonglinux               |
| Router (radixtree) | Khớp request với route rule theo URI, Host, Header, Method. Hot-reload không restart congdonglinux       |
| Plugin Runtime     | Chain các plugin theo thứ tự: rewrite → access → proxy → header_filter → body_filter → log congdonglinux   |
| Balancer           | LB sang upstream: round-robin, least-conn, consistent-hashing, EWMA congdonglinux                        |
| SSL/TLS Engine     | Dynamic certificate loading, không cần restart khi update cert apisix.apache                              |

#### Control Plane (quản lý cấu hình)
| Thành phần            | Vai trò                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| etcd                  | Config store phân tán (Raft consensus). APISIX watch etcd để nhận config thay đổi realtime congdonglinux   |
| Admin API (port 9180) | REST API để CRUD Route/Upstream/Plugin/Consumer/SSL. Không expose public congdonglinux                   |
| APISIX Dashboard      | Web UI optional, gọi Admin API bên dưới congdonglinux                                                    |

##### Các đối tượng cấu hình chính
Route       → rule khớp request (URI, method, host, header)
Upstream    → nhóm backend servers + LB algorithm + health check
Service     → tái sử dụng plugin config cho nhiều Route
Plugin      → logic xử lý gắn vào Route/Service/Global
Consumer    → định danh end-user (API key, JWT, HMAC)
SSL         → cert/key mapping theo SNI
Global Rule → plugin áp dụng cho tất cả request

```bash
# Luồng Dữ Liệu
Admin → POST /apisix/admin/routes  (Control Plane)
              │
              ▼
           etcd  ←── APISIX watch (subscribe)
                            │
                            ▼
              Data Plane hot-reload vào memory
              (không restart, không drop connection)
              │
              ▼
Client request → Data Plane dùng config đã load để xử lý
```

---
### 2. Chức năng

#### Traffic Management
- Load Balancing: round-robin, least-conn, consistent-hash
- Health check: active (HTTP probe) + passive (based on response)
- Circuit breaker, retry, timeout, traffic splitting (canary/A-B)
- Rate limiting theo IP/User/Route

#### Security
- TLS termination (dynamic cert, không restart)
- Authentication: API Key, JWT, HMAC, Basic Auth, OAuth2, OIDC
- IP whitelist/blacklist (ip-restriction plugin)
- Request validation (block malformed request trước khi vào backend)

#### Observability
- Access log → file/Kafka/HTTP
- Metrics → Prometheus exporter (port 9091)
- Tracing → OpenTelemetry / Zipkin / SkyWalking

#### Extensibility
- Plugin tự viết bằng Lua, Go (plugin runner), WASM
- Standalone mode: load config từ YAML file, không cần etcd

---
### 3. Hoạt Động Cơ Bản
```
# Lifecycle một request S3 qua APISIX:
Client
  │  HTTPS :443
  ▼
APISIX (Data Plane)
  ├─ 1. TLS Termination
  ├─ 2. Router khớp Route (Host: s3.tenant.vn)
  ├─ 3. Plugin chain thực thi:
  │      ├─ ip-restriction (whitelist)
  │      ├─ limit-req (rate limit)
  │      └─ proxy-rewrite (nếu cần)
  ├─ 4. Balancer chọn RGW instance (health check pass)
  └─ 5. Forward HTTP → RGW :3950
              │
              ▼
        RGW-Data (Lua script, S3 API)
```
```
# Luồng config change (dynamic, zero-downtime):
Admin → Admin API (9180) → etcd → APISIX watch → hot-reload vào memory
                                    (không restart, không drop connection)
```
---
### 4. Hình thức triển khai
Deployment mode của APISIX:
- Traditional: data plane + control plane chung 1 process (đơn giản, phù hợp production nhỏ)
- Decoupled: tách CP (port 9180) và DP riêng — phù hợp khi scale nhiều DP node
- Standalone: không cần etcd, load config từ YAML — phù hợp môi trường simple/airgap

| Hình thức | Ưu điểm | Nhược điểm | Phù hợp với S3RGW |
| --------- | ------- | ---------- | ----------------- |
| On-host (bare metal/VM)   | Hiệu suất tốt nhất, không overhead container. Full control resource. Debug dễ | Cài đặt thủ công, upgrade phức tạp, không portable                      | ✅ Phù hợp nhất với môi trường on-prem Ceph |
| Container (Docker/Podman) | Dễ deploy, isolate, rollback image. Phù hợp Podman trên Ceph node             | Cần quản lý volume, network mode, sys limit                             | ✅ Tốt nếu đang dùng Podman (Cephadm stack) |
| Kubernetes                | Auto-scaling, self-healing, Ingress Controller native. Ecosystem GitOps tốt   | Phức tạp, overhead K8s control plane, không có sẵn K8s trong S3RGW HLD  | ❌ Over-engineering cho bài toán này        |
| GitOps (CI/CD)            | Config as code, audit trail, rollback dễ. Kết hợp với bất kỳ runtime nào      | Cần pipeline CI/CD infrastructure (ArgoCD/Flux), học thêm toolchain uit | ⚠️ Nên kết hợp sau khi ổn định on-host     |

---
### 5. Mô hình triển khai

#### Pattern 1 - Chỉ RGW-data đi qua APISIX

##### Inbound/Outbound listen port
```
                    APISIX Process
              ┌─────────────────────────────────────┐
CLIENT ───────┼──► :443 (Inbound)                   │   - Nhận S3 request từ client, public port 443 cho client thấy
(S3 API/      │  Data Plane Listener                │   - TLS terminate tại đây, APISIX giải mã rồi forward xuống
 App/SDK)     │     → nhận S3 request               │
              │         │                           │
              │         ▼                           │
              │  [Plugin chain]                     │
              │  Upstream backend (Outbound) ───────┼──► RGW-Data :3950
              │  [Balancer]                         │   - APISIX chỉ là client, RGW-data mới là server.
              │                                     │   - APISIX không listen :3950
ADMIN ────────┼─► :9180 (Inbound)                   │   
(curl/UI)     │  Control Plane - Admin API Listener │   - Không có traffic S3 đi qua đây
              │     → CRUD Route/Upstream           │   - Chỉ dùng để cấu hình APISIX (CRUD đối tượng) 
              │     → ghi vào etcd                  │   - Phải IP whitelist nghiêm ngặt, không expose internet
              │                                     │
              └─────────────────────────────────────┘
```

##### Luồng S3 request tới RGW-data thông qua APISIX
```
[S3 Client]
    │
    │ 1. HTTPS :443 → APISIX (Inbound)
    ▼
[APISIX Data Plane]
    │  TLS decrypt
    │  Router match (Host header)
    │  Plugin chain (auth, rate-limit...)
    │  Balancer chọn node
    │
    │ 2. HTTP :3950 → RGW-Data (Outbound)
    ▼
[RGW-Data Process]
    │  Lua prerequest
    │  S3 logic (IAM, bucket, object)
    │  Ghi RADOS
    │  Lua postrequest
    │
    │ 3. HTTP Response → APISIX
    ▼
[APISIX Data Plane]
    │  Plugin log/header
    │
    │ 4. HTTPS Response → Client
    ▼
[S3 Client]
```

##### Phương án 1.A — 3 VM (gộp APISIX vào Ceph VM)
```
DC1: VM Ceph DC1 + APISIX DC1  (chung VM)
DC2: VM Ceph DC2 + APISIX DC2  (chung VM)
DC1: VM APISIX ZG              (VM riêng)

CLIENT ──► APISIX ──► RGW-Data :3950
                      RGW-Sync :3951 ◄──────────────────► RGW-Sync :3951
                      (bypass APISIX, direct DC1 ↔ DC2)
```

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│              DC1                │     │              DC1                │ 
│ VM-1: [APISIX DC1] + [Ceph DC1] │     │ VM-2: [APISIX DC1] + [Ceph DC2] │
├─────────────────────────────────┤     ├─────────────────────────────────┤
│ [APISIX DC1]                    |     │ [APISIX DC2]                    |
│   └─ Listen :443 ───────┐       │     │   └─ Listen :443 ───────┐       │
│                         │       │     │                         │       │
│ [Ceph DC1]              │       │     │ [Ceph DC2]              │       │
│   ├─ RGW-Data :3950 ◄───┘       │     │   ├─ RGW-Data :3950 ◄───┘       │
│   └─ RGW-Sync :3951 ◄───────────┼─────┼──►└─ RGW-Sync :3951             │
│   (direct, bypass APISIX)       │     │   (direct, bypass APISIX)       │
│    s3.dc1-rgw-tenant.vn         │     │    s3.dc2-rgw-tenant.vn         │
├─────────────────────────────────┤     └─────────────────────────────────┘
│ VM-3: [APISIX ZG]               │
│   └─ Listen :443                │
│    s3.rgw-tenant.vn             │
└─────────────────────────────────┘

⚠️  RESOURCE CONTENTION
    Ceph OSD spike IO/CPU
    └──► APISIX latency tăng
    └──► S3 API SLA bị ảnh hưởng
```

|          | Đánh giá                                                                                              |
| -------- | ----------------------------------------------------------------------------------------------------- |
| ✅ Ưu    | Đơn giản nhất, ít VM, triển khai nhanh. RGW-Sync không đổi luồng, không rủi ro timeout                |
| ❌ Nhược | Resource contention Ceph + APISIX chung VM. Blast radius lớn. Ceph OSD spike ảnh hưởng S3 API latency |
| ⚠️ Lưu ý | Cần cgroup/CPU pinning để APISIX và Ceph không tranh nhau. OS-level sysctl tuning phức tạp            |

##### Phương án 1.B — 5 VM (tách hoàn toàn)
```
DC1: VM Ceph DC1
DC2: VM Ceph DC2
DC1: VM APISIX DC1
DC2: VM APISIX DC2
DC1: VM APISIX ZG
```

```
┌──────────────────────────────┐    ┌──────────────────────────────┐
│            DC1               │    │            DC2               │ 
├──────────────────────────────┤    ├──────────────────────────────┤ 
│ VM-2: [APISIX DC1]           │    │ VM-4: [APISIX DC2]           │ 
│   └─ Listen :443 ───────┐    │    │   └─ Listen :443 ───────┐    │
├─────────────────────────┼────┤    ├─────────────────────────┼────┤
│ VM-1: [Ceph DC1]        │    │    │ VM-3: [Ceph DC2]        │    │
│   ├─ RGW-Data :3950 ◄───┘    │    │   ├─ RGW-Data :3950 ◄───┘    │
│   └─ RGW-Sync :3951 ◄────────┼────┼──►└─ RGW-Sync :3951          │
│   (direct, bypass APISIX)    │    │   (direct, bypass APISIX)    │
│    s3.dc1-rgw-tenant.vn      │    │    s3.dc2-rgw-tenant.vn      │
├──────────────────────────────┤    └──────────────────────────────┘
│ VM-5: [APISIX ZG]            │
│   └─ Listen :443             │
│    s3.rgw-tenant.vn          │
└──────────────────────────────┘

✅  FULL ISOLATION
    Ceph OSD spike    → không ảnh hưởng APISIX
    APISIX traffic     → không tranh CPU/RAM Ceph
    Blast radius nhỏ  → VM die độc lập
```
|          | Đánh giá                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------- |
| ✅ Ưu    | Full isolation Ceph vs APISIX. Blast radius nhỏ. Tune độc lập. Scale độc lập. RGW-Sync không bị ảnh hưởng bởi APISIX config |
| ❌ Nhược | +2 VM → chi phí cao hơn. Thêm network hop APISIX VM → Ceph VM (~0.1–0.3ms, chấp nhận được)                                 |
| ⚠️ Lưu ý | RGW-Sync đi thẳng DC1 → DC2, không đi qua APISIX → không có log/monitor Sync traffic tập trung                              |


#### Pattern 2 - Cả 2 RGW-data và RGW-sync đều đi qua APISIX

##### [Luồng S3 request tới RGW-data thông qua APISIX tương tự với Pattern 1](#luồng-s3-request-tới-rgw-data-thông-qua-apisix)
##### Luồng sync giữa RGW-sync của 2 DC thông qua APISIX
```
[RGW-Sync DC1]
    │
    │ 1. Outbound HTTP → APISIX DC2 :3951
    │    (DC1 chủ động kết nối sang DC2)
    ▼
[APISIX DC2] Listener :3951
    │  Router match port :3951
    │  Plugin chain:
    │   ├─ ip-restriction  (chỉ cho phép IP DC1)
    │   ├─ proxy-rewrite   (nếu cần)
    │   └─ log             (audit sync traffic)
    │  Balancer → RGW-Sync DC2
    │
    │ 2. Outbound HTTP → RGW-Sync DC2 :3951
    ▼
[RGW-Sync DC2 :3951]
    │  Xử lý sync metadata/data
    │  Ghi vào RADOS Pool DC2
    │
    │ 3. Response → APISIX DC2
    ▼
[APISIX DC2]
    │
    │ 4. Response → RGW-Sync DC1
    ▼
[RGW-Sync DC1]
    (xác nhận sync thành công)
```

##### Phương án 2.A — 3 VM (gộp APISIX vào Ceph VM)
```
CLIENT ──► APISIX :443  ──► RGW-Data :3950
DC1 APISIX :3951 ◄──────────────────────────► DC2 APISIX :3951
                                                      │
                                               RGW-Sync :3951
```
```
┌───────────────────────────────────┐     ┌───────────────────────────────────┐
│                DC1                │     │                DC2                │
│ VM-1: [APISIX DC1] + [Ceph DC1]   │     │ VM-2: [APISIX DC2] + [Ceph DC2]   │
├───────────────────────────────────┤     ├───────────────────────────────────┤
│ [APISIX DC1]                      |     │ [APISIX DC2]                      |
│   ├─ Listen :3951 ◄───────────────┼─────┼─► ├─ Listen :3951                 │
│   │   └─► RGW-Sync                |     │   │   └─► RGW-Sync                |
│   └─ Listen :443                  │     │   └─ Listen :443                  │
│       └─► RGW-Data                |     │       └─► RGW-Data                |
│                                   │     │                                   │
│ [Ceph DC1]                        │     │ [Ceph DC2]                        │
│   ├─ RGW-Data :3950               │     │   ├─ RGW-Data :3950               │
│   └─ RGW-Sync :3951               │     │   └─ RGW-Sync :3951               │
│   (direct, bypass APISIX)         │     │   (direct, bypass APISIX)         │
│    s3.dc1-rgw-tenant.vn           │     │    s3.dc2-rgw-tenant.vn           │
├───────────────────────────────────┤     └───────────────────────────────────┘
│ VM-3: [APISIX ZG]                 │
│   └─ Listen :443                  │
│    s3.rgw-tenant.vn               │
└───────────────────────────────────┘
```
|          | Đánh giá                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------- |
| ✅ Ưu    | Ít VM nhất. Tập trung log cả Data + Sync traffic. Có thể monitor và alert Sync failure qua APISIX                           |
| ❌ Nhược | Worst case: Ceph + APISIX chung VM + Sync traffic đi qua APISIX → 3 workload nặng tranh tài nguyên. <br> APISIX die = mất cả S3 API lẫn Sync → data inconsistency nguy hiểm |
| ⚠️ Lưu ý | Sync là long-lived streaming connection, cần proxy_read_timeout = 0 hoặc rất lớn. <br> Nếu APISIX restart → Sync bị ngắt → RGW-Sync phải reconnect → có window inconsistency tạm thời |

##### Phương án 2.B — 5 VM (tách hoàn toàn)
```
┌──────────────────────────────┐      ┌──────────────────────────────┐
│            DC1               │      │            DC2               │ 
├──────────────────────────────┤      ├──────────────────────────────┤ 
│ VM-2: [APISIX DC1]           │      │ VM-4: [APISIX DC2]           │ 
│   ├─ Listen :3951 ◄──────────┼──────┼─► ├─ Listen :3951            │
│   │   └─► RGW-Sync           |      │   │   └─► RGW-Sync           |
│   └─ Listen :443             │      │   └─ Listen :443             │
│       └─► RGW-Data           |      │       └─► RGW-Data           |
├──────────────────────────────┤      ├──────────────────────────────┤
│ VM-1: [Ceph DC1]             │      │ VM-3: [Ceph DC2]             │
│   ├─ RGW-Data :3950          │      │   ├─ RGW-Data :3950          │
│   └─ RGW-Sync :3951          │      │   └─ RGW-Sync :3951          │
│   (direct, bypass APISIX)    │      │   (direct, bypass APISIX)    │
│    s3.dc1-rgw-tenant.vn      │      │    s3.dc2-rgw-tenant.vn      │
├──────────────────────────────┤      └──────────────────────────────┘
│ VM-5: [APISIX ZG]            │
│   └─ Listen :443             │
│    s3.rgw-tenant.vn          │
└──────────────────────────────┘
```
|          | Đánh giá                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------- |
| ✅ Ưu    | Full isolation. Log + monitor tập trung cả Data lẫn Sync. Ceph không bị ảnh hưởng khi APISIX config thay đổi. Scale và tune hoàn toàn độc lập   |
| ❌ Nhược | +2 VM. Sync đi qua thêm 1 hop APISIX → cần tune timeout cẩn thận. Config phức tạp hơn 1.B                                   |
| ⚠️ Lưu ý | Cần 2 Route riêng trên APISIX DC1/DC2: <br> - Route :443 (timeout ngắn, rate-limit) <br> - Route :3951 (timeout = 0, không rate-limit). <br> APISIX restart phải graceful để Sync không bị ngắt đột ngột |


Ma trận so sánh
|                     |   1.A   |   1.B   |   2.A   |       2.B         |
| ------------------- | ------- | ------- | ------- | ---------------   |
| VM count            |    3    |    5    |    3    |        5          |
| Chi phí             |   Thấp  | Cao hơn |   Thấp  |     Cao hơn       |
| Resource isolation  |    ❌   |   ✅    |  ❌ ❌  |       ✅         |
| Blast radius        |    ❌   |   ✅    |  ❌❌   |       ✅         |
| Sync visibility     |    ❌   |   ❌    |   ✅    |       ✅         |
| Sync risk           |   Thấp  |  Thấp   |   Cao   |    Trung bình    | 
| Config phức tạp      |   Thấp  |  Trung  |  Trung  |       Cao        |
| SLA S3 API          |    ⚠️   |   ✅    |   ❌    |       ✅         |
| Production-ready    |    ❌   |   ✅    |   ❌    | ⚠️ với điều kiện |


###### Lưu Ý Khi Cho RGW-Sync Đi Qua APISIX: Khuyến nghị thực tế là tạo 2 Route riêng biệt trên APISIX cho :443 (S3 client) và :3951 (RGW-Sync) với timeout config khác nhau — S3 client timeout ngắn, Sync timeout rất dài hoặc disabled.
```
                  Lợi ích                    Rủi ro / Lưu ý
                ────────────────────        ─────────────────────────────
RGW-Data :443   TLS termination tập         Cần tune buffer lớn cho
                trung, rate limit,          multipart upload
                health check, log

RGW-Sync :3951  Centralized mTLS,           ⚠️ Sync là long-lived HTTP
                log sync traffic,           connection, stream lớn
                health check Sync           → cần tắt timeout hoặc
                instance                    set rất cao trên APISIX
                                            (proxy_read_timeout lớn)

                                            ⚠️ Sync failure = data
                                            inconsistency → cần alert
                                            riêng cho port :3951
```

|       | Đánh giá |
| ----- | -------- |
| Ưu    | Isolation hoàn toàn: Ceph và APISIX không tranh tài nguyên. Blast radius nhỏ: APISIX die không ảnh hưởng Ceph và ngược lại. Tune độc lập: mỗi VM tối ưu riêng cho workload của nó. Scale độc lập: tăng APISIX VM khi traffic tăng, không cần đụng Ceph. Dễ troubleshoot, log rõ ràng theo role |
| Nhược | +2 VM → chi phí hardware/OS/vận hành cao hơn. Network hop giữa APISIX VM → Ceph VM (thêm ~0.1-0.3ms latency trong nội DC, chấp nhận được) |

| Trường hợp                               | Khuyến nghị                                     |
| ---------------------------------------- | ----------------------------------------------- |
| Dev / Lab / PoC, tài nguyên hạn chế      | 1.A — Đơn giản, nhanh, không cần HA Sync        |
| Production on-prem, SLA S3 API cao       | 1.B — Isolation tốt, Sync direct không rủi ro   |
| Cần audit log toàn bộ traffic kể cả Sync  | 2.B — Visibility đầy đủ nhưng cần tune cẩn thận |
| Cần giảm VM nhưng vẫn monitor Sync       | 2.A — Chỉ chấp nhận ở môi trường non-critical   |
| Tương lai scale thêm APISIX instance     | 1.B hoặc 2.B                                    |

> **Khuyến nghị**:
> - Nên chọn Phương án B (5 VM) cho môi trường production S3RGW vì:
>   - Ceph OSD workload rất bất thường (rebalance, recovery, backfill đột ngột spike IO/CPU) — nếu gộp chung sẽ gây latency spike cho APISIX, ảnh hưởng trực tiếp SLA S3 API
>   - APISIX cần được tune riêng (connection pool, worker process, ulimit) — conflict với Ceph sysctl
>   - Khi cần mở rộng (thêm APISIX instance sau etcd cluster), phương án B linh hoạt hơn nhiều
>   - Phù hợp với nguyên tắc No Single Point of Failure trong HLD v8 — không để gateway và storage cùng một failure domain
> - Nếu chi phí VM thực sự là constraint, phương án thỏa hiệp là gộp APISIX ZG vào DC1 với APISIX DC1 (giảm xuống 4 VM), vì APISIX ZG chỉ làm nhiệm vụ routing nhẹ hơn so với APISIX DC1/DC2 xử lý data traffic.
> Lý do:
>   - RGW-Sync là critical path — mọi gián đoạn Sync dẫn đến data inconsistency giữa DC1 và DC2. Không nên đặt thêm điểm lỗi (APISIX) vào luồng này trừ khi có yêu cầu rõ ràng về visibility
>   - Ceph OSD workload bất thường (rebalance, recovery) → cần tách hoàn toàn khỏi APISIX để bảo vệ SLA S3 API
>   - Nếu sau này cần monitor Sync traffic, có thể chuyển lên 2.B bằng cách thêm Route :3951 vào APISIX DC1/DC2 mà không cần thay đổi topology VM


---
# LLD
## Cài đặt APISIX
```
apisix/
├── docker-compose.yml
├── apisix_conf/
│   └── config.yaml
├── dashboard_conf/
│   └── conf.yaml
├── logs/
│   ├── apisix/
│   └── dashboard/
└── etcd_data/
```
```
Host machine
├── 127.0.0.1        (loopback - chỉ host thấy)
├── 172.17.0.1       (docker0 - gateway của default bridge)
└── 172.20.0.1       (apisix-net - gateway của network apisix tạo - đây là gateway IP của Docker network apisix-net — tức là địa chỉ của host nhìn từ trong container thuộc network đó.)
         |           Container trong apisix-net muốn gọi ra host → phải dùng 172.20.0.1, không phải 127.0.0.1.
         │
         ├── apisix container        172.20.x.x
         └── apisix-dashboard container  172.20.x.x
```

Cài đặt etcd: <https://apisix.apache.org/docs/apisix/installation-guide/#installing-etcd>
```bash
ETCD_VERSION='3.5.4'
wget https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
tar -xvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz && cd etcd-v${ETCD_VERSION}-linux-amd64 && sudo cp -a etcd etcdctl /usr/bin/
nohup etcd >/tmp/etcd.log 2>&1 &
```
1. Kiểm tra process etcd đang chạy
```bash
ps aux | grep etcd
```

2. Kiểm tra service (systemd)
```bash
systemctl status etcd
```

3. Kiểm tra port 2379/2380 có đang được lắng nghe không
```bash
ss -tlnp | grep -E '2379|2380'
# hoặc
netstat -tlnp | grep -E '2379|2380'
```

4. Kiểm tra binary etcd có tồn tại không
```bash
which etcd
etcd --version
```

5. Kiểm tra Docker container etcd cũ (nếu ai đó đã chạy bằng Docker trước đó)
```bash
docker ps -a | grep etcd

# Lấy docker bridge IP
ip addr show docker0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
#  Thường là 172.17.0.1
```

```bash
sudo nano /etc/systemd/system/etcd.service
```
``` config
[Unit]
Description=etcd key-value store
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/etcd \
  --name=etcd0 \
  --data-dir=/var/lib/etcd \
  --listen-client-urls=http://0.0.0.0:2379 \
  --advertise-client-urls=http://172.25.216.164:2379 \
  --listen-peer-urls=http://0.0.0.0:2380 \
  --initial-advertise-peer-urls=http://172.25.216.164:2380 \
  --initial-cluster=etcd0=http://172.25.216.164:2380
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
sudo systemctl status etcd
sudo systemctl daemon-reload && sudo systemctl restart etcd
```

# Verify etcd nghe trên bridge IP
ss -tlnp | grep 2379

-----bỏ
# Tạo user root trước (bắt buộc phải có root user trước khi enable auth)
etcdctl --endpoints=http://127.0.0.1:2379 user add root # Nhập password khi được hỏi, ví dụ: StrongEtcdPassword123!
# Gán role root cho user root
etcctl --endpoints=http://127.0.0.1:2379 user grant-role root root
# Bật auth
etcdctl --endpoints=http://127.0.0.1:2379 auth enable
# Verify
etcdctl --endpoints=http://127.0.0.1:2379 --user=root:root endpoint health
-----bỏ


# Xem owner hiện tại
ls -la ~/apisix/logs/

# Fix permission — APISIX container chạy với user nobody (uid 65534)
sudo chown -R 65534:65534 ~/apisix/logs/

# Hoặc đơn giản hơn
chmod -R 777 ~/apisix/logs/


# Sinh admin key mạnh
openssl rand -hex 32

# Khởi động APISIX
docker compose up

# Kiểm tra trạng thái
docker compose ps
docker compose logs -f apisix
docker compose down && docker compose up -d



# Test từ trong dashboard container
docker exec apisix-dashboard sh -c "wget -qO- http://etcd-host:2379/health"
# Xem etcd-host resolve về IP nào
docker exec apisix-dashboard sh -c "cat /etc/hosts | grep etcd"
cat ~/apisix/logs/apisix/error.log
ss -tlnp | grep 2379
docker exec apisix-dashboard cat /etc/hosts
docker exec apisix-dashboard sh -c "ip route | grep default"
docker network inspect apisix_apisix-net | grep Gateway
# Verify
docker exec apisix-dashboard cat /etc/hosts | grep etcd
# Expected: 172.20.0.1   etcd-host

docker logs apisix-dashboard --tail=10


# Test Admin API
curl -H "X-API-KEY: your-admin-api-key-here" http://localhost:9180/apisix/admin/routes


# Dùng APISIX proxy Dashboard (elegant nhất)
# Tạo route trên APISIX trỏ vào dashboard:
curl -X PUT http://172.25.216.164:9180/apisix/admin/routes/dashboard -H "X-API-KEY: 1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c" -H "Content-Type: application/json" -d '{"uri": "/*","host": "api6.lab.thuyldx","upstream": {"type": "roundrobin","nodes": {"apisix-dashboard:9000": 1}}}'

# -> ruy cập: http://<IP_VM> với Host header dashboard.yourdomain.com

# Hoặc nếu chưa có domain, dùng path-based:
curl -X PUT http://172.25.216.164:9180/apisix/admin/routes/dashboard -H "X-API-KEY: 1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c" -H "Content-Type: application/json" -d '{"uri": "/*","upstream": {"type": "roundrobin","nodes": {"apisix-dashboard:9000": 1}},"plugins": {"ip-restriction": {"whitelist": ["<IP_máy_bạn>/32"]}}}'


## Cấu hình cho Zonegroup HCM VIP Active-Passive
# Tạo upstream cho 
curl -X PUT http://172.25.216.164:9180/apisix/admin/upstreams/hcm-rgw -H "X-API-KEY: 1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c" -H "Content-Type: application/json" -d '{"name": "hcm-rgw","desc": "Ceph RGW HCM - Active/Passive","type": "roundrobin","nodes": [{"host": "172.25.216.241","port": 3950,"weight": 1,"priority": 1},{"host": "172.25.216.186","port": 3950,"weight": 1,"priority": 0}],"checks": {"active": {"type": "http","http_path": "/","healthy": {"interval": 5,"successes": 2},"unhealthy": {"interval": 5,"http_failures": 3}}},"scheme": "http"}'

Client (HTTPS 443)
    ↓
APISIX :443  ← SSL termination tại đây
    ↓  HTTP nội bộ
RGW HCM-1 :3950  (primary)
RGW HCM-4 :3950  (backup, chỉ active khi HCM-1 down)

# Tạo Route path-style + virtual-host
# Path-style (s3.hcm.lab.thuyldx/bucket):
curl -X PUT http://172.25.216.164:9180/apisix/admin/routes/hcm-s3-path -H "X-API-KEY: 1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c" -H "Content-Type: application/json" -d '{"name": "hcm-s3-path-style","uri": "/*","host": "s3.hcm.lab.thuyldx","upstream_id": "hcm-rgw"}'

# Virtual-host style (bucket.s3.hcm.lab.thuyldx):
curl -X PUT http://172.25.216.164:9180/apisix/admin/routes/hcm-s3-vhost -H "X-API-KEY: 1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c" -H "Content-Type: application/json" -d '{"name": "hcm-s3-virtual-host","uri": "/*","host": "*.s3.hcm.lab.thuyldx","upstream_id": "hcm-rgw"}'



# Upload cert wildcard *.hcm.lab.thuyldx lên APISIX
curl -s http://ap6.lab.thuyldx:9180/apisix/admin/ssls \
  -H "X-API-KEY: ldf5f679c8669b7f06a8db95016ec37" \
  -X POST -d "{
    \"cert\": $(jq -Rs . < ~/thuyldx.crt),
    \"key\": $(jq -Rs . < ~/thuyldx.key),
    \"snis\": [
      \"*.thuyldx\",
      \"*.lab.thuyldx\",
      \"*.hcm.lab.thuyldx\",
      \"*.hni.lab.thuyldx\"
    ]
  }" | jq .

hoặc
# Tạo file JSON payload trước
jq -n \
  --rawfile cert ~/thuyldx.crt \
  --rawfile key ~/thuyldx.key \
  '{
    cert: $cert,
    key: $key,
    snis: ["*.thuyldx", "*.lab.thuyldx", "*.hcm.lab.thuyldx", "*.hni.lab.thuyldx"]
  }' > /tmp/ssl_payload.json

# Kiểm tra payload trước
cat /tmp/ssl_payload.json | jq '{snis, cert_preview: (.cert | .[0:50])}'

# Upload lên APISIX
curl -s http://ap6.lab.thuyldx:9180/apisix/admin/ssls \
  -H "X-API-KEY: ldf5f679c8669b7f06a8db95016ec37" \
  -H "Content-Type: application/json" \
  -X POST \
  -d @/tmp/ssl_payload.json | jq .



# Cập nhật DNS record - Thêm dòng và tăng Serial
sudo nano /etc/bind/zones/db.lab.thuyldx
sudo named-checkzone lab.thuyldx /etc/bind/zones/db.lab.thuyldx && sudo rndc reload
dig ap6.lab.thuyldx +short




# Tạo Upstream + Route cho s3.hcm.lab.thuyldx:
APISIX="http://172.25.216.164:9180/apisix/admin"
KEY="1df5f679c8669b7f06a8db95016ec377742810717fa2d7d1d4221e877f12a84c"

# Bước 2a — Tạo Upstream
curl -s $APISIX/upstreams/upstream-rgw-hcm \
  -H "X-API-KEY: $KEY" \
  -H "Content-Type: application/json" \
  -X PUT -d '{
    "id": "upstream-rgw-hcm",
    "name": "RGW HCM Zonegroup",
    "type": "roundrobin",
    "scheme": "http",
    "nodes": {
      "172.25.216.241:3950": 1,
      "172.25.216.186:3950": 1
    }
  }' | jq .

# Bước 2b — Tạo Route
curl -s $APISIX/routes/route-s3-hcm \
  -H "X-API-KEY: $KEY" \
  -H "Content-Type: application/json" \
  -X PUT -d '{
    "id": "route-s3-hcm",
    "name": "S3 HCM Zonegroup",
    "host": "s3.hcm.lab.thuyldx",
    "uri": "/*",
    "methods": ["GET","PUT","POST","DELETE","HEAD","OPTIONS","PATCH"],
    "upstream_id": "upstream-rgw-hcm"
  }' | jq .


  # 1. Test HTTPS kết nối được không
curl -vk https://s3.hcm.lab.thuyldx 2>&1 | grep -E "SSL|subject|issuer|HTTP|< "

# 2. Test S3 API - list buckets
curl -sk https://s3.hcm.lab.thuyldx \
  -H "Host: s3.hcm.lab.thuyldx" | head -20

# 3. Test bằng s3cmd (thay access/secret key)
s3cmd --host=s3.hcm.lab.thuyldx \
      --host-bucket="%(bucket)s.s3.hcm.lab.thuyldx" \
      --access_key=YOUR_ACCESS_KEY \
      --secret_key=YOUR_SECRET_KEY \
      --no-ssl-check \
      ls

# 4. Test bằng AWS CLI
aws s3 ls \
  --endpoint-url https://s3.hcm.lab.thuyldx \
  --no-verify-ssl
