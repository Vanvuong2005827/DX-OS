# Chiến lược kiểm thử

## 1. Test pyramid

```text
          E2E ít nhưng quan trọng
       Contract / integration tests
          Unit tests số lượng lớn
```

Không dùng E2E để thay unit test business rule.

## 2. Backend

### Unit

- state transition table;
- permission/ownership policy;
- amount/budget/attachment validation;
- idempotency behavior;
- error mapping;
- AI tool precondition.

### Integration

- repository với PostgreSQL thật;
- migration từ database rỗng;
- transaction rollback;
- outbox claim/retry;
- JWT/JWKS validation;
- Nextcloud/RAGFlow client qua mock server.

### Concurrency

- hai manager approve cùng `version`;
- hai finance approval cạnh tranh cùng budget;
- cùng idempotency key gửi lặp;
- worker claim cùng outbox event.

## 3. Frontend

- component/store unit tests;
- Reactive Form validation;
- role guard/visibility;
- Problem Details hiển thị đúng;
- 409 reload flow;
- upload progress/error;
- accessibility smoke.

## 4. Contract

- OpenAPI lint.
- Backend response khớp schema.
- Angular typed client build được.
- Event payload khớp JSON Schema.
- Breaking change detector trong CI.

## 5. E2E nghiệp vụ

| ID | Kịch bản | Kết quả |
|---|---|---|
| E2E-01 | employee login và tạo draft | phiếu có mã/version |
| E2E-02 | submit thiếu báo giá khi vượt ngưỡng | 422, không đổi status |
| E2E-03 | requester tự approve | 403 và security audit |
| E2E-04 | manager khác department approve | 403 |
| E2E-05 | manager request changes | requester sửa/resubmit được |
| E2E-06 | manager + finance approve | status APPROVED, budget cập nhật |
| E2E-07 | stale version approve | 409, không double event |
| E2E-08 | dashboard | KPI khớp fixture |
| E2E-09 | RAG | trả lời có nguồn hoặc từ chối |
| E2E-10 | Agent execution chưa approve | bị chặn |
| E2E-11 | Agent approved execute | một tool call, audit đầy đủ |
| E2E-12 | restore demo | môi trường trở lại fixture |

`E2E-01` hiện được tự động hóa bằng:

```powershell
.\scripts\Test-OIDCLogin.ps1 -TestProcurement
```

Script đăng nhập bằng Authorization Code + PKCE, tạo draft, kiểm tra list theo employee scope và đọc
detail. Credential/token chỉ tồn tại trong `data/runtime` đã Git ignore và được xóa sau khi test.

## 6. Security tests

- token thiếu/sai issuer/audience/expired.
- role tampering và IDOR.
- SQL injection/filter abuse.
- XSS trong title/comment/file name.
- upload file type/size/path traversal.
- SSRF qua URL integration.
- prompt injection từ document.
- rate limit AI/tool.
- secret/token không xuất hiện trong log.

## 7. Data quality

- request code unique.
- required fields not null.
- item/request FK.
- total bằng tổng line total.
- accepted status/currency.
- approved request có approved timestamp.
- dashboard reconciliation với operational data.

## 8. Performance baseline

Cho MVP lab:

- list/detail API p95 dưới 500 ms với dataset demo;
- transition p95 dưới 1 s, không tính downstream async;
- dashboard query có timeout và index;
- upload giới hạn theo policy;
- AI latency báo riêng, không áp SLA web thông thường.

Con số phải được đo trên môi trường Demo/UAT và ghi hardware/dataset.

## 9. CI gate

- Workflow: `.github/workflows/ci.yml`.
- Push được kiểm tra trên `main` và `feat/enterprise-operations-review`; pull request được kiểm tra khi nhắm vào `main`. Có thể chạy thủ công bằng `workflow_dispatch`.
- Go backend: `gofmt`, `go vet`, unit/race test và build ba command `api`, `worker`, `migrate`.
- Angular frontend: Prettier, unit test, production build và audit production dependencies ở mức `high`.
- OpenAPI: Spectral lint cho `contracts/openapi/dx-os-v1.yaml`.
- Docusaurus: typecheck, build/kiểm tra liên kết và dependency audit ở mức `high`.

Hai advisory `image-size` gián tiếp từ Docusaurus được ghi rõ trong `docs-site/audit-ci.jsonc` vì chưa có bản vá npm có thể cài đặt. Ngoại lệ hết hạn ngày **2026-10-31**; advisory mới hoặc ngoại lệ quá hạn vẫn làm CI thất bại. Chỉ tài liệu và ảnh do repository kiểm soát được phép đi qua bước build này.

Trước khi push, chạy các lệnh tương đương gate định dạng:

```powershell
docker run --rm -v "${PWD}/backend:/src" -w /src golang:1.26.6-alpine gofmt -l .
npm --prefix frontend run format:check
npm --prefix docs-site run audit:ci
```

E2E full chạy trên integration/nightly; smoke subset chạy khi deploy.

## 10. Bằng chứng nghiệm thu

Mỗi giai đoạn lưu:

- test report;
- log/correlation ID mẫu đã redact;
- ảnh/video demo;
- version/commit/image digest;
- migration version;
- known issues và risk acceptance.
