# OVC – Oracle Version Control

> Tài liệu đặc tả (specs) – phiên bản 0.1 (draft)
> Ngôn ngữ: Go · Database: Oracle 19c · Git server: GitLab của công ty

---

## 1. Bối cảnh & mục tiêu

### 1.1 Bối cảnh

- Dự án làm theo hướng **DB-first**: API chỉ gọi procedure/package trên Oracle, phần lớn business logic nằm trong PL/SQL.
- Code trên DB rất lớn nhưng **không có version control**: sửa trực tiếp trên DB, đè code của nhau, không biết ai sửa gì, khi nào, không rollback được.

### 1.2 Mục tiêu

1. Toàn bộ source của mỗi schema Oracle được quản lý trong **một repository GitLab**.
2. Dev làm việc giống code thông thường: **pull → sửa local → push → Merge Request → review → merge**.
3. **Conflict được phát hiện và giải quyết trên file**, không còn chuyện đè code trên DB.
4. **Mọi thay đổi DB chỉ đi qua một luồng duy nhất**: merge vào branch → GitLab pipeline deploy.
5. Biết được **ai, sửa gì, khi nào, deploy lên đâu** (audit).
6. Phát hiện được thay đổi ngoài luồng (**drift**) giữa DB và Git.

### 1.3 Ngoài phạm vi (giai đoạn đầu)

- Version control cho **dữ liệu** (chỉ quản lý cấu trúc + code, trừ dữ liệu master nhỏ nếu cần ở phase sau).
- Tự động sinh script `ALTER TABLE` từ việc so sánh 2 phiên bản bảng.
- Giao diện web riêng (dùng giao diện GitLab cho MR/review/pipeline).
- Tự động rollback thay đổi cấu trúc bảng.

---

## 2. Thuật ngữ

| Thuật ngữ | Ý nghĩa |
|---|---|
| **OVC CLI** (`ovc`) | Công cụ dòng lệnh chạy trên máy dev, 1 file binary. |
| **OVC Server** (`ovc-server`) | Server trung gian: nhận yêu cầu từ CLI, thao tác Git/GitLab, đọc DB. |
| **Schema repo** | Repository GitLab chứa source của **một** schema Oracle. |
| **Workspace** | Thư mục local trên máy dev chứa bản sao của một schema repo. |
| **Base SHA** | Commit mà workspace được lấy về; dùng làm gốc khi merge 3 chiều. |
| **Code object** | Object tạo lại được bằng `CREATE OR REPLACE`: package, procedure, function, view, trigger, type. |
| **Migration** | Script chạy **một lần** để đổi cấu trúc/dữ liệu (`ALTER TABLE`, data fix...). |
| **Drift** | Khác biệt giữa DB thực tế và source trên Git. |
| **Environment (env)** | Một DB đích: `dev`, `test`, `prod`... |

---

## 3. Nguyên tắc thiết kế

1. **Git là nguồn sự thật (source of truth).** DB chỉ là nơi chạy code được deploy từ Git.
2. **Một luồng thay đổi duy nhất.** DB chỉ thay đổi qua GitLab pipeline. Không sửa tay, kể cả hotfix.
3. **Phân tách quyền:**
   - OVC Server **chỉ có quyền ĐỌC** DB (export, diff, drift).
   - **Chỉ GitLab pipeline giữ key GHI DB** (deploy).
   - Dev **không có quyền DDL** trên DB dùng chung và không cần biết mật khẩu DB.
4. **Dev chỉ nói chuyện với OVC Server.** CLI không kết nối trực tiếp GitLab hay Oracle.
5. **Phân quyền theo GitLab.** Ai có quyền gì trên repo GitLab thì có quyền tương ứng trên OVC.
6. **Không tự giải conflict thay người dùng.** OVC chỉ merge khi sạch; có conflict thì trả về cho dev xử lý.

---

## 4. Kiến trúc tổng thể

```
 Developer
    │  ovc login / clone / pull / status / diff / push / mr ...
    ▼
┌──────────┐  HTTPS + token user  ┌────────────────────┐   git + GitLab API   ┌──────────────────┐
│ OVC CLI  │ ───────────────────► │ OVC Server         │ ───────────────────► │ GitLab công ty   │
│ (local)  │ ◄─────────────────── │ (trung gian)       │ ◄─────────────────── │ repo / MR / CI   │
└──────────┘   source, kết quả    │ - git mirror repo  │                      │                  │
                                  │ - metadata DB      │                      │  Runner          │
                                  │ - audit log        │                      │  (giữ key ghi DB)│
                                  │ - DB READ-ONLY     │                      └──────────────────┘
                                  └────────────────────┘                               │
                                          │ SQL*Net (chỉ đọc:                          │ SQL*Net (ghi:
                                          │ init, drift)                               │ deploy)
                                          ▼                                            ▼
                                  ┌──────────────────────────────────────────────────────────┐
                                  │              Oracle 19c (dev / test / prod)               │
                                  └──────────────────────────────────────────────────────────┘
```

### 4.1 Thành phần

| Thành phần | Vai trò | Ghi chú |
|---|---|---|
| `ovc` (CLI) | Thao tác của dev: lấy source, theo dõi thay đổi local, push, tạo MR. | 1 binary cho Windows/Linux/macOS. Không cần Go, Git hay Oracle Client. |
| `ovc-server` | API cho CLI; quản lý bản mirror các schema repo; merge/commit/push lên GitLab; export DDL; kiểm tra drift; audit. | Chạy trong Docker trên VM nội bộ. Cần `git` cài trên server. |
| Metadata DB của OVC | Lưu danh sách schema đã đăng ký, cấu hình DB, audit log, lock. | PostgreSQL (SQLite cho môi trường dev/test của tool). |
| GitLab công ty | Repo, MR, review, phân quyền, CI/CD. | Không sửa GitLab, chỉ dùng API + CI. |
| GitLab Runner | Chạy pipeline deploy. Giữ credential ghi DB trong CI/CD variables. | Chạy `ovc deploy` (cùng binary CLI, chế độ deploy). |
| Oracle 19c | DB đích. | Có schema quản trị `OVC_ADMIN` để lưu lịch sử deploy. |

### 4.2 Luồng tổng quát

```
ovc clone/pull ──► OVC Server ──► GitLab (lấy source)
dev sửa file local
ovc push ───────► OVC Server ──► kiểm tra + merge + commit ──► GitLab (feature branch)
ovc mr ─────────► OVC Server ──► GitLab API (tạo MR)
review + merge trên GitLab
merge vào develop/release/main ──► Pipeline ──► ovc deploy ──► Oracle (dev/test/prod)
định kỳ: ovc-server drift ──► so DB với Git ──► báo cáo
```

---

## 5. Cấu trúc schema repo

Mỗi schema Oracle = 1 project GitLab, đặt trong group cấu hình sẵn (vd `db/`), tên project = tên schema viết thường (vd `db/hr`).

```
hr/
├── ovc.yaml                    # cấu hình repo (schema, quy tắc deploy)
├── .gitlab-ci.yml              # include template pipeline chung
├── tables/                     # snapshot cấu trúc bảng hiện tại (tham chiếu)
│   └── EMPLOYEES.sql
├── indexes/
│   └── EMPLOYEES_IX1.sql
├── constraints/                # FK tách riêng để tránh phụ thuộc thứ tự
│   └── EMPLOYEES.sql
├── sequences/
│   └── EMPLOYEES_SEQ.sql
├── views/
│   └── V_EMPLOYEE_INFO.sql
├── materialized_views/
├── types/
│   ├── T_EMP_REC.tps           # type spec
│   └── T_EMP_REC.tpb           # type body
├── packages/
│   ├── PKG_EMPLOYEE.pks        # package spec
│   └── PKG_EMPLOYEE.pkb        # package body
├── procedures/
│   └── P_SYNC_EMPLOYEE.prc
├── functions/
│   └── F_GET_SALARY.fnc
├── triggers/
│   └── TRG_EMPLOYEES_BIU.trg
├── synonyms/
│   └── SYN_DEPT.sql
├── grants/
│   └── grants.sql              # grant mà schema này cấp cho schema khác
├── jobs/                       # DBMS_SCHEDULER jobs (phase sau)
└── migrations/
    ├── V20261006_1530__add_column_email.sql
    └── V20261007_0910__fix_status_data.sql
```

### 5.1 Quy tắc đặt tên file

- Tên file = **tên object đúng như Oracle lưu** (thường là chữ HOA). Object có tên quoted chữ thường giữ nguyên tên.
- Mỗi file chứa **đúng 1 object** (riêng package/type tách spec và body thành 2 file).
- Phần mở rộng theo loại object:

| Loại object | Thư mục | Đuôi file |
|---|---|---|
| TABLE | `tables/` | `.sql` |
| INDEX | `indexes/` | `.sql` |
| CONSTRAINT (FK) | `constraints/` | `.sql` (gom theo bảng) |
| SEQUENCE | `sequences/` | `.sql` |
| VIEW | `views/` | `.sql` |
| MATERIALIZED VIEW | `materialized_views/` | `.sql` |
| TYPE / TYPE BODY | `types/` | `.tps` / `.tpb` |
| PACKAGE / PACKAGE BODY | `packages/` | `.pks` / `.pkb` |
| PROCEDURE | `procedures/` | `.prc` |
| FUNCTION | `functions/` | `.fnc` |
| TRIGGER | `triggers/` | `.trg` |
| SYNONYM | `synonyms/` | `.sql` |

### 5.2 Định dạng nội dung file

- Encoding **UTF-8**, xuống dòng **LF**, không có khoảng trắng thừa cuối dòng.
- **Không ghi tên schema** trong DDL (`CREATE OR REPLACE PACKAGE PKG_EMPLOYEE`, không phải `HR.PKG_EMPLOYEE`) để deploy được sang schema/DB khác.
- Code object bắt đầu bằng `CREATE OR REPLACE` và kết thúc bằng `/`.
- Không chứa storage/tablespace/segment attributes (để không lệch giữa các môi trường).

### 5.3 Hai loại thay đổi

| Loại | Áp dụng cho | Cách deploy | Khi file thay đổi |
|---|---|---|---|
| **Repeatable** | package, procedure, function, view, trigger, type, synonym | Chạy lại toàn bộ file (`CREATE OR REPLACE`) | Deploy lại file đó |
| **Versioned (migration)** | table, index, constraint, sequence, dữ liệu | Chạy **1 lần duy nhất**, ghi lại vào `OVC_ADMIN.OVC_MIGRATION_LOG` | Không được sửa migration đã deploy |

- Thư mục `tables/`, `indexes/`, `constraints/`, `sequences/` là **snapshot tham chiếu** cấu trúc hiện tại: dev cập nhật cùng MR với migration, pipeline **không chạy** các file này (trừ khi dựng môi trường mới từ đầu – `ovc deploy --bootstrap`). Drift check dùng snapshot này để so với DB.
- Tên migration: `V<yyyyMMdd>_<HHmm>__<mô_tả>.sql`, sinh bằng `ovc new migration <mô_tả>`. Dùng timestamp thay vì số thứ tự để 2 dev không đặt trùng tên.

### 5.4 File `ovc.yaml`

```yaml
schema: HR
version: 1
deploy:
  allow_drop: false          # xoá file code object có sinh DROP không
  recompile: true            # compile lại object invalid sau deploy
  fail_on_new_invalid: true  # pipeline fail nếu deploy làm phát sinh object invalid mới
exclude:                     # object không quản lý
  - "SYS_*"
  - "BIN$*"
  - "MLOG$_*"
  - "RUPD$_*"
  - "TMP_*"
```

---

## 6. Workspace local của dev

```
~/work/hr/
├── .ovc/
│   ├── workspace.yaml   # schema, server URL, branch đang làm
│   ├── state.json       # base SHA + hash từng file tại base
│   └── base/            # bản gốc tại base SHA (để diff/restore offline)
├── packages/...
└── ...
```

- `state.json` ghi `base_sha`, `branch` và `sha256` của từng file → CLI tự biết file nào **thêm / sửa / xoá** mà không cần Git trên máy dev.
- Cấu hình chung của user: `~/.ovc/config.yaml` (server URL, token lưu trong OS keychain; fallback file mã hoá quyền 0600).

---

## 7. OVC CLI – đặc tả lệnh

Cú pháp chung: `ovc <lệnh> [tham số] [--flag]`. Mọi lệnh hỗ trợ `--json` (output máy đọc) và `--verbose`.

### 7.1 Tài khoản & cấu hình

| Lệnh | Mô tả |
|---|---|
| `ovc login [--server URL]` | Đăng nhập bằng GitLab Personal Access Token (scope `api`, `write_repository`). Server xác thực token với GitLab rồi lưu vào keychain. |
| `ovc logout` | Xoá token local. |
| `ovc whoami` | Hiện user, server, phiên bản. |
| `ovc version` | Phiên bản CLI và server; cảnh báo nếu không tương thích. |
| `ovc update` | Tự tải bản CLI mới nhất từ server và thay thế binary. |

### 7.2 Schema & workspace

| Lệnh | Mô tả |
|---|---|
| `ovc schemas` | Liệt kê schema đã đăng ký mà user có quyền. |
| `ovc init <SCHEMA> --db <alias>` | (Quyền Maintainer/Admin) Export toàn bộ schema từ DB, tạo project GitLab, commit baseline lên `main`, tạo `develop`. |
| `ovc clone <SCHEMA> [dir] [--branch B]` | Lấy source về workspace mới (mặc định branch `develop`). |
| `ovc branch` | Hiện branch hiện tại và danh sách branch. |
| `ovc branch create <tên>` | Tạo feature branch trên GitLab từ `develop` (tên chuẩn hoá: `feature/<user>/<tên>`), chuyển workspace sang branch đó. |
| `ovc switch <branch>` | Chuyển workspace sang branch khác (từ chối nếu có thay đổi chưa push, trừ `--force`). |

### 7.3 Làm việc với thay đổi

| Lệnh | Mô tả |
|---|---|
| `ovc status` | Liệt kê file thêm / sửa / xoá / đang conflict so với base. |
| `ovc diff [file...]` | Hiện diff so với base. |
| `ovc restore <file...>` | Huỷ thay đổi local, khôi phục file từ base. |
| `ovc pull` | Lấy thay đổi mới nhất của branch, **merge 3 chiều** với thay đổi local (xem §9.3). |
| `ovc pull --from develop` | Đưa thay đổi mới của `develop` vào feature branch hiện tại. |
| `ovc push -m "<message>"` | Gửi thay đổi lên server → server kiểm tra, commit, push lên branch hiện tại (xem §9.4). |
| `ovc new migration <mô_tả>` | Sinh file migration theo chuẩn tên. |
| `ovc resolve <file...>` | Đánh dấu file conflict đã được giải quyết. |

### 7.4 Merge Request & theo dõi

| Lệnh | Mô tả |
|---|---|
| `ovc mr create [--target develop] [--title T]` | Tạo MR từ branch hiện tại. In ra link MR. |
| `ovc mr list` | Các MR đang mở của schema. |
| `ovc mr status` | Trạng thái MR của branch hiện tại (review, pipeline). |
| `ovc log [file]` | Lịch sử commit (của cả repo hoặc 1 file). |
| `ovc drift --env <env>` | Yêu cầu server so DB môi trường `env` với branch tương ứng, hiện kết quả. |

### 7.5 Chế độ deploy (chỉ dùng trong pipeline)

| Lệnh | Mô tả |
|---|---|
| `ovc deploy --env <env>` | Deploy commit hiện tại lên DB. Đọc thông tin kết nối từ biến môi trường CI. Xem §10. |
| `ovc deploy --env <env> --dry-run` | Chỉ in kế hoạch deploy (file nào, thứ tự nào), không chạy. |
| `ovc deploy --env <env> --bootstrap` | Dựng schema trống từ snapshot + toàn bộ migration (môi trường mới). |

### 7.6 Mã thoát (exit code)

| Code | Ý nghĩa |
|---|---|
| 0 | Thành công |
| 1 | Lỗi chung |
| 2 | Sai tham số |
| 3 | Chưa đăng nhập / token hết hạn |
| 4 | Không đủ quyền |
| 5 | Conflict |
| 6 | Lỗi deploy (lỗi SQL / object invalid) |
| 7 | Phát hiện drift |

---

## 8. OVC Server – đặc tả API

- REST/JSON qua HTTPS, prefix `/api/v1`.
- Xác thực: header `Authorization: Bearer <GitLab PAT>`. Server gọi GitLab `GET /user` để xác thực, cache kết quả 5 phút.
- Phân quyền: lấy access level của user trên project GitLab tương ứng:
  - **Reporter+**: clone, pull, status, log, drift.
  - **Developer+**: push lên feature branch, tạo MR.
  - **Maintainer+** hoặc admin OVC: `init` schema.
- Header `X-OVC-Client-Version` bắt buộc; server trả `426 Upgrade Required` nếu CLI quá cũ.

| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/api/v1/version` | Phiên bản server, phiên bản CLI tối thiểu, link tải CLI. |
| GET | `/api/v1/me` | Thông tin user hiện tại. |
| GET | `/api/v1/schemas` | Danh sách schema user có quyền. |
| POST | `/api/v1/schemas` | Init schema mới `{schema, db_alias}`. Chạy bất đồng bộ, trả `job_id`. |
| GET | `/api/v1/jobs/{id}` | Trạng thái job dài (init, drift). |
| GET | `/api/v1/schemas/{s}/branches` | Danh sách branch. |
| POST | `/api/v1/schemas/{s}/branches` | Tạo branch `{name, from}`. |
| GET | `/api/v1/schemas/{s}/snapshot?ref=<branch>` | Tải toàn bộ source (`tar.gz`) + header `X-OVC-Commit`. |
| POST | `/api/v1/schemas/{s}/sync` | Pull có merge: gửi `{branch, base_sha, changes[]}`, nhận kết quả merge (§9.3). |
| POST | `/api/v1/schemas/{s}/push` | Push: gửi `{branch, base_sha, message, changes[]}` (§9.4). |
| GET | `/api/v1/schemas/{s}/log?ref=&path=` | Lịch sử commit. |
| POST | `/api/v1/schemas/{s}/merge-requests` | Tạo MR `{source, target, title, description}`. |
| GET | `/api/v1/schemas/{s}/merge-requests` | Danh sách MR đang mở. |
| POST | `/api/v1/schemas/{s}/drift` | Chạy drift check `{env}` → `job_id`. |
| GET | `/api/v1/audit?schema=&user=&from=&to=` | Tra cứu audit log (admin). |
| GET | `/healthz`, `/readyz` | Health check. |

`changes[]` gồm các phần tử `{path, action: add|modify|delete, content_base64, sha256}`.

Lỗi trả về dạng chuẩn:

```json
{ "error": { "code": "CONFLICT", "message": "2 file bị conflict", "details": { "files": ["packages/PKG_EMPLOYEE.pkb"] } } }
```

---

## 9. Các luồng chi tiết

### 9.1 Init schema (`ovc init HR --db dev`)

1. Kiểm tra quyền admin/maintainer, kiểm tra schema chưa được đăng ký.
2. Server kết nối DB bằng user **read-only** `OVC_READER` của alias `dev`.
3. Lấy danh sách object từ `DBA_OBJECTS` (lọc theo `exclude`, bỏ object do hệ thống sinh: `GENERATED='Y'`, recycle bin, LOB/IOT segment...).
4. Với mỗi object, lấy DDL bằng `DBMS_METADATA.GET_DDL` với transform:
   - `SEGMENT_ATTRIBUTES = FALSE`, `STORAGE = FALSE`, `TABLESPACE = FALSE`
   - `EMIT_SCHEMA = FALSE`, `SQLTERMINATOR = TRUE`, `PRETTY = TRUE`
   - `CONSTRAINTS_AS_ALTER = TRUE`, `REF_CONSTRAINTS = FALSE` (FK lấy riêng vào `constraints/`)
5. Chuẩn hoá nội dung (LF, bỏ khoảng trắng cuối dòng, chuẩn `CREATE OR REPLACE`, kết thúc `/`).
6. Export song song (worker pool, mặc định 8 kết nối) để xử lý schema hàng nghìn object.
7. Tạo project GitLab `db/<schema>`, push commit baseline lên `main`, tạo `develop`, thêm `ovc.yaml`, `.gitlab-ci.yml`.
8. Bật protected branch cho `main`, `develop`, `release/*` qua GitLab API.
9. Ghi schema vào metadata DB, ghi audit.

### 9.2 Clone

1. CLI gọi `snapshot?ref=develop` → nhận `tar.gz` + commit SHA.
2. Giải nén vào thư mục, ghi `.ovc/workspace.yaml`, `.ovc/state.json` (`base_sha`, hash từng file), copy bản gốc vào `.ovc/base/`.

### 9.3 Pull (merge 3 chiều)

Dev có thể đang sửa dở khi pull. Merge thực hiện **trên server** (dùng `git merge-file`/`git merge` thật):

- **base** = repo tại `base_sha`
- **ours** = file local của dev (CLI gửi các file đã thay đổi)
- **theirs** = đầu branch hiện tại trên GitLab

Kết quả cho từng file:

| Trường hợp | Kết quả |
|---|---|
| Chỉ phía remote đổi | Cập nhật file local. |
| Chỉ local đổi | Giữ nguyên. |
| Cả hai đổi, không chồng dòng | Merge tự động, file local nhận bản đã merge. |
| Cả hai đổi, chồng dòng | **Conflict**: file local được ghi kèm marker `<<<<<<< local` / `=======` / `>>>>>>> remote`, đánh dấu `conflicted` trong `state.json`. |
| Một bên xoá, một bên sửa | Conflict loại delete/modify, dev chọn giữ hay xoá. |

Sau pull: `base_sha` = đầu branch mới. Dev sửa file conflict, chạy `ovc resolve <file>`. **`ovc push` bị chặn khi còn file conflict hoặc còn marker conflict trong file.**

### 9.4 Push

1. CLI kiểm tra local: không còn conflict, có ít nhất 1 thay đổi.
2. CLI gửi `{branch, base_sha, message, changes[]}`.
3. Server **khoá repo theo branch** (tránh 2 push đồng thời trên cùng branch).
4. Server kiểm tra (validation):
   - Branch không phải `main`/`develop`/`release/*` (protected – chỉ thay đổi qua MR).
   - Đường dẫn file nằm trong thư mục cho phép, đúng đuôi theo §5.1.
   - Tên object trong nội dung khớp tên file và loại object khớp thư mục.
   - Không chứa tên schema cứng (cảnh báo), không chứa marker conflict.
   - UTF-8 hợp lệ, kích thước file ≤ giới hạn (mặc định 5 MB).
   - Migration đã có trong branch đích thì **không được sửa/xoá**.
5. Nếu `base_sha` ≠ đầu branch: server thử merge 3 chiều như §9.3.
   - Sạch → tiếp tục.
   - Có conflict → **từ chối** (`409 CONFLICT`, exit code 5), trả danh sách file; dev chạy `ovc pull` rồi xử lý.
6. Tạo commit:
   - **Author** = tên/email của user (lấy từ GitLab).
   - Push lên GitLab **bằng chính token của user** → GitLab tự áp quyền, lịch sử đúng người.
7. Trả về commit SHA mới, CLI cập nhật `state.json` và `.ovc/base/`.
8. Ghi audit: user, schema, branch, commit, danh sách file.

### 9.5 Merge Request & review

- `ovc mr create` → server gọi GitLab API tạo MR (mặc định target `develop`).
- Pipeline `validate` chạy trên MR (§10.4).
- Review, comment, approve, merge trên **giao diện GitLab**.
- Conflict ở mức MR (feature branch cũ hơn `develop`): dev chạy `ovc pull --from develop`, xử lý conflict, `ovc push`.

### 9.6 Drift check

1. Server export DDL từ DB của `env` (như §9.1 bước 3–5) vào thư mục tạm.
2. So với branch tương ứng của env (`dev`→`develop`, `test`→`release/*` mới nhất, `prod`→`main`).
3. Báo cáo: object **chỉ có trên DB**, **chỉ có trên Git**, **khác nội dung** (kèm diff).
4. Chạy theo lịch (scheduled pipeline hoặc cron trong server, mặc định mỗi đêm) và theo yêu cầu (`ovc drift`).
5. Có drift → gửi thông báo (email/webhook chat) cho maintainer của schema, ghi audit.

---

## 10. Deploy pipeline (GitLab CI)

### 10.1 Ánh xạ branch → môi trường

| Branch | Môi trường | Kích hoạt |
|---|---|---|
| `develop` | `dev` | Tự động khi merge |
| `release/*` | `test` | Tự động khi merge |
| `main` | `prod` | **Manual** – chỉ nhóm được phép bấm |
| `hotfix/*` → MR vào `main` | `prod` | Như `main` |

### 10.2 Template pipeline

Lưu trong repo chung `db/ovc-ci-templates`, các schema repo chỉ `include`:

```yaml
# .gitlab-ci.yml trong mỗi schema repo
include:
  - project: db/ovc-ci-templates
    file: /oracle-deploy.yml
```

```yaml
# db/ovc-ci-templates/oracle-deploy.yml
stages: [validate, deploy]

default:
  image: registry.company.local/ovc/ovc-cli:1

validate:
  stage: validate
  script:
    - ovc deploy --env dev --dry-run
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

.deploy:
  stage: deploy
  script:
    - ovc deploy --env $OVC_ENV
  resource_group: $CI_PROJECT_NAME-$OVC_ENV   # mỗi env chỉ 1 deploy tại 1 thời điểm

deploy-dev:
  extends: .deploy
  variables: { OVC_ENV: dev }
  environment: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy-test:
  extends: .deploy
  variables: { OVC_ENV: test }
  environment: test
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^release\//

deploy-prod:
  extends: .deploy
  variables: { OVC_ENV: prod }
  environment: prod
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### 10.3 Credential deploy

- Lưu trong **GitLab CI/CD variables** (group `db`), loại **masked + protected**, theo env:
  - `OVC_DB_DEV_DSN`, `OVC_DB_DEV_USER`, `OVC_DB_DEV_PASSWORD`
  - `OVC_DB_TEST_...`, `OVC_DB_PROD_...`
- Biến của `prod` chỉ được đọc bởi protected branch `main`.
- Dùng **proxy authentication** của Oracle: user `OVC_DEPLOYER` kết nối thay mặt schema owner (`OVC_DEPLOYER[HR]`) → object được tạo đúng owner mà không cần quyền `CREATE ANY ...`.

### 10.4 Thuật toán `ovc deploy`

1. Kết nối DB qua proxy user, đọc commit đã deploy gần nhất từ `OVC_ADMIN.OVC_DEPLOY_LOG` (schema + env).
2. Tính danh sách file thay đổi: `git diff --name-status <last_sha> <CI_COMMIT_SHA>`. Chưa từng deploy → yêu cầu `--bootstrap` hoặc `--baseline` (đánh dấu commit hiện tại là đã deploy mà không chạy gì – dùng cho DB đã có sẵn code lúc init).
3. Ghi bản ghi deploy trạng thái `RUNNING`, lấy lock (`DBMS_LOCK`) theo schema để chặn deploy song song.
4. Chạy **migration** chưa có trong `OVC_MIGRATION_LOG`, theo thứ tự tên; mỗi migration thành công ghi log ngay. Lỗi → dừng.
5. Chạy **code object** đã thêm/sửa theo thứ tự phụ thuộc:
   1. sequences, synonyms
   2. type spec
   3. package spec
   4. function, procedure
   5. view, materialized view
   6. type body, package body
   7. trigger
   8. grants
6. File code object bị xoá: nếu `allow_drop: true` sinh `DROP`, ngược lại **fail** và yêu cầu migration drop tường minh.
7. Compile lại object invalid (`DBMS_UTILITY.COMPILE_SCHEMA(schema, compile_all => FALSE)`).
8. So danh sách object invalid trước/sau deploy; phát sinh invalid mới và `fail_on_new_invalid: true` → pipeline **fail** (exit 6), in lỗi từ `ALL_ERRORS`.
9. Cập nhật `OVC_DEPLOY_LOG` (`SUCCESS`/`FAILED`, commit, pipeline id, người merge, thời gian, danh sách file).
10. Gửi kết quả về OVC Server (webhook, tuỳ chọn) để lưu audit và comment vào MR.

> Lưu ý: DDL trên Oracle tự commit, không rollback được theo transaction. Deploy lỗi giữa chừng → sửa và deploy tiếp (roll-forward). Code object có thể quay về bằng cách deploy lại commit cũ; migration cần script đảo ngược viết tay.

### 10.5 Bảng quản trị trên mỗi DB đích (schema `OVC_ADMIN`)

```sql
CREATE TABLE OVC_DEPLOY_LOG (
  ID            NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  SCHEMA_NAME   VARCHAR2(128) NOT NULL,
  ENV           VARCHAR2(30)  NOT NULL,
  COMMIT_SHA    VARCHAR2(40)  NOT NULL,
  PREV_SHA      VARCHAR2(40),
  STATUS        VARCHAR2(20)  NOT NULL,      -- RUNNING / SUCCESS / FAILED
  PIPELINE_ID   VARCHAR2(50),
  TRIGGERED_BY  VARCHAR2(128),
  STARTED_AT    TIMESTAMP DEFAULT SYSTIMESTAMP,
  FINISHED_AT   TIMESTAMP,
  DETAIL        CLOB                          -- JSON: file đã chạy, lỗi
);

CREATE TABLE OVC_MIGRATION_LOG (
  SCHEMA_NAME   VARCHAR2(128) NOT NULL,
  MIGRATION     VARCHAR2(255) NOT NULL,
  CHECKSUM      VARCHAR2(64)  NOT NULL,       -- phát hiện migration bị sửa sau khi chạy
  DEPLOY_ID     NUMBER REFERENCES OVC_DEPLOY_LOG(ID),
  APPLIED_AT    TIMESTAMP DEFAULT SYSTIMESTAMP,
  CONSTRAINT PK_OVC_MIGRATION_LOG PRIMARY KEY (SCHEMA_NAME, MIGRATION)
);
```

---

## 11. Bảo mật & bắt buộc "một luồng"

### 11.1 User Oracle

| User | Quyền | Ai giữ |
|---|---|---|
| `OVC_READER` | `CREATE SESSION`, `SELECT_CATALOG_ROLE`, `SELECT ANY DICTIONARY` (đọc `DBA_OBJECTS`, `DBMS_METADATA`) | OVC Server |
| `OVC_DEPLOYER` | `CREATE SESSION`; được proxy vào các schema quản lý (`ALTER USER HR GRANT CONNECT THROUGH OVC_DEPLOYER`); quyền ghi `OVC_ADMIN` | GitLab CI variables |
| User của dev | Không có quyền DDL trên schema được quản lý ở env dùng chung | Dev |

### 11.2 Chặn DDL ngoài luồng

Trigger cấp DB trên mỗi env, triển khai 2 giai đoạn:

1. **Giai đoạn quan sát** (2–4 tuần): chỉ ghi log vào `OVC_ADMIN.OVC_DDL_AUDIT`.
2. **Giai đoạn chặn**: raise lỗi.

```sql
CREATE OR REPLACE TRIGGER OVC_ADMIN.TRG_OVC_GUARD_DDL
BEFORE DDL ON DATABASE
DECLARE
  v_proxy   VARCHAR2(128) := SYS_CONTEXT('USERENV', 'PROXY_USER');
  v_managed NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_managed
    FROM OVC_ADMIN.OVC_MANAGED_SCHEMA
   WHERE SCHEMA_NAME = ora_dict_obj_owner;

  IF v_managed > 0 AND NVL(v_proxy, '-') <> 'OVC_DEPLOYER' THEN
    RAISE_APPLICATION_ERROR(-20001,
      'OVC: thay đổi schema ' || ora_dict_obj_owner ||
      ' phải đi qua GitLab pipeline');
  END IF;
END;
/
```

Ở giai đoạn quan sát, thay `RAISE_APPLICATION_ERROR` bằng `INSERT` vào `OVC_DDL_AUDIT` (dùng `PRAGMA AUTONOMOUS_TRANSACTION` trong một procedure riêng).

### 11.3 GitLab

- Protected branch `main`, `develop`, `release/*`: không ai push trực tiếp, chỉ merge qua MR.
- MR cần tối thiểu 1 approval (cấu hình theo group).
- Job `deploy-prod` là manual, giới hạn bằng protected environment.

### 11.4 OVC Server

- Chỉ chạy HTTPS (TLS do reverse proxy nội bộ cấp).
- Không lưu token của user lâu dài: chỉ dùng trong request, cache kết quả xác thực (không cache token gốc) 5 phút.
- Credential `OVC_READER` và service token GitLab (dùng cho init/drift) lấy từ biến môi trường / secret manager, không ghi trong file cấu hình.
- Audit mọi thao tác ghi (init, push, MR, drift).
- Rate limit theo user.

---

## 12. Metadata DB của OVC Server

PostgreSQL, các bảng chính:

| Bảng | Nội dung |
|---|---|
| `db_alias` | Alias DB (`dev`, `test`, `prod`): DSN, tham chiếu secret của `OVC_READER`. |
| `schema_repo` | Schema đã đăng ký: tên schema, alias DB gốc, GitLab project id, đường dẫn mirror, ngày init. |
| `env_mapping` | Ánh xạ env ↔ branch theo schema (mặc định như §10.1). |
| `job` | Job bất đồng bộ (init, drift): trạng thái, tiến độ, kết quả. |
| `drift_report` | Kết quả drift check theo lần chạy. |
| `audit_log` | user, hành động, schema, branch, commit, thời gian, IP, chi tiết JSON. |

---

## 13. Cấu hình

### 13.1 Server (`ovc-server.yaml` + biến môi trường)

```yaml
listen: ":8080"
public_url: https://ovc.company.local
gitlab:
  url: https://gitlab.company.local
  group: db
  service_token_env: OVC_GITLAB_TOKEN
storage:
  mirror_dir: /var/lib/ovc/repos
metadata_db:
  dsn_env: OVC_METADATA_DSN
databases:
  dev:  { dsn: "dbdev.company.local:1521/DEVPDB",  user: OVC_READER, password_env: OVC_DB_DEV_PASSWORD }
  test: { dsn: "dbtest.company.local:1521/TESTPDB", user: OVC_READER, password_env: OVC_DB_TEST_PASSWORD }
  prod: { dsn: "dbprod.company.local:1521/PRODPDB", user: OVC_READER, password_env: OVC_DB_PROD_PASSWORD }
export:
  workers: 8
drift:
  schedule: "0 1 * * *"
cli:
  min_version: 0.1.0
  download_dir: /var/lib/ovc/cli
```

### 13.2 CLI (`~/.ovc/config.yaml`)

```yaml
server: https://ovc.company.local
editor: code
```

---

## 14. Công nghệ

| Hạng mục | Lựa chọn |
|---|---|
| Ngôn ngữ | Go ≥ 1.23 |
| CLI framework | `github.com/spf13/cobra` |
| HTTP server | `net/http` + `github.com/go-chi/chi/v5` |
| Oracle driver | `github.com/sijms/go-ora/v2` (thuần Go, không cần Instant Client) |
| GitLab client | `gitlab.com/gitlab-org/api/client-go` |
| Git trên server | Gọi lệnh `git` (≥ 2.40) – đầy đủ merge/rebase hơn thư viện thuần Go |
| Metadata DB | PostgreSQL 15+ (`github.com/jackc/pgx/v5`) |
| Cấu hình | YAML (`gopkg.in/yaml.v3`) + biến môi trường |
| Keychain | `github.com/zalando/go-keyring` |
| Log | `log/slog` (JSON) |
| Đóng gói | Binary cho `windows/amd64`, `linux/amd64`, `darwin/arm64`, `darwin/amd64`; Docker image cho server và CLI (dùng trong CI) |

---

## 15. Cấu trúc source của dự án OVC

```
oracle_version_control/
├── cmd/
│   ├── ovc/                  # main của CLI
│   └── ovc-server/           # main của server
├── internal/
│   ├── cli/                  # các lệnh cobra
│   ├── client/               # HTTP client gọi OVC Server
│   ├── workspace/            # state.json, base, tính thay đổi local
│   ├── server/               # HTTP handlers, middleware auth
│   ├── auth/                 # xác thực + phân quyền qua GitLab
│   ├── gitops/               # mirror, merge 3 chiều, commit, push
│   ├── gitlab/               # wrapper GitLab API
│   ├── oracle/
│   │   ├── export/           # DBMS_METADATA, chuẩn hoá DDL
│   │   ├── deploy/           # thuật toán deploy, thứ tự, recompile
│   │   └── drift/
│   ├── layout/               # quy tắc thư mục/đuôi file/tên object
│   ├── store/                # metadata DB (PostgreSQL)
│   └── audit/
├── sql/
│   └── install/              # script tạo OVC_ADMIN, user, trigger
├── ci-templates/             # template GitLab CI cho schema repo
├── deploy/                   # Dockerfile, docker-compose
├── docs/
└── specs.md
```

---

## 16. Yêu cầu phi chức năng

| Hạng mục | Yêu cầu |
|---|---|
| Hiệu năng | Init schema 5.000 object < 10 phút; clone < 30 giây; push < 5 giây (không tính thời gian mạng GitLab). |
| Đồng thời | Hỗ trợ ≥ 50 dev dùng cùng lúc; khoá theo branch khi push. |
| Sẵn sàng | Server không có trạng thái quan trọng ngoài metadata DB và mirror (mirror dựng lại được từ GitLab). |
| Sao lưu | Backup metadata DB hằng ngày. Source chính nằm ở GitLab (đã có backup của công ty). |
| Dự phòng | OVC ngừng hoạt động → dev vẫn có thể dùng git trực tiếp với GitLab; pipeline deploy không phụ thuộc OVC Server. |
| Quan sát | Log JSON, endpoint `/metrics` (Prometheus), audit log tra cứu được. |
| Tương thích | Oracle 19c; GitLab ≥ 16; Windows 10+, macOS 12+, Linux. |

---

## 17. Lộ trình

| Phase | Nội dung | Kết quả |
|---|---|---|
| **0 – Nền tảng** | Khung repo Go, CI build binary, script `sql/install` (OVC_ADMIN, user, bảng log). | Build được `ovc` và `ovc-server`. |
| **1 – Export & Init** | `oracle/export`, `ovc init`, tạo project GitLab + baseline. | Mỗi schema có repo trên GitLab. |
| **2 – Deploy pipeline** | `ovc deploy` (migration + code object + recompile + log), CI template. | Merge vào branch → tự deploy. **Từ đây đã có "một luồng".** |
| **3 – Workspace CLI** | `login`, `clone`, `status`, `diff`, `restore`, `pull`, `push`, `branch`, `mr`. | Dev làm việc qua OVC thay vì sửa trên DB. |
| **4 – Drift & guard** | Drift check định kỳ + báo cáo; trigger DDL chế độ quan sát → chặn. | Phát hiện và chặn sửa ngoài luồng. |
| **5 – Hoàn thiện** | `ovc update`, audit UI/tra cứu, comment kết quả vào MR, metrics, ký số binary Windows. | Sẵn sàng dùng rộng. |

> Phase 1–2 đã đủ để dev dùng **git trực tiếp** với GitLab nếu cần; phase 3 mang lại trải nghiệm qua OVC.

---

## 18. Vấn đề còn mở

1. **Test trước khi push:** dev không có quyền DDL → cần một schema/PDB **sandbox** cho từng dev, hay chấp nhận kiểm tra tại env `dev` sau khi merge vào `develop`?
2. **Kiểm tra compile trong MR:** có cấp một schema "scratch" để pipeline `validate` compile thử code trước khi merge không?
3. **Phụ thuộc giữa các schema:** schema A gọi package của schema B → thứ tự deploy giữa các repo (multi-project pipeline?).
4. **Object ngoài phạm vi:** DB link, scheduler job, queue (AQ), context, policy (VPD) – quản lý hay loại trừ?
5. **Dữ liệu master/cấu hình:** có cần version control cho một số bảng tham số không?
6. **Đăng nhập:** chỉ dùng Personal Access Token hay hỗ trợ OAuth device flow của GitLab?
7. **Xác nhận** `go-ora` hỗ trợ proxy authentication (`OVC_DEPLOYER[HR]`) trên 19c; nếu không, phương án thay thế là `ALTER SESSION SET CURRENT_SCHEMA` + quyền `ANY`.
8. **Quy ước branch** (`develop` / `release/*` / `main`) có khớp quy trình hiện tại của công ty không?
