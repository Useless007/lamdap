# Lamdap M0: วางฐานบน RR stack — แผนงาน

**เป้าหมายหลัก:** เมื่อจบ M0 ทั้งสองคนรันโปรเจกต์ที่สร้างจาก RR stack ได้ในเครื่องของตัวเอง, ทุก PR ถูกตรวจด้วยคำสั่งของ stack, หน้ารายการ issue หน้าแรกทำงานบนสัญญาของ sync client ทั้งที่ยังไม่มี engine จริง, เจ้าของเลือกหน้าตาแล้ว และแอปเปิดได้จากอินเทอร์เน็ตผ่าน Cloudflare Tunnel จากเครื่องของเจ้าของ

**Goal:** วางโครง repo, สัญญาสองชั้น และทางขึ้นระบบจริง เพื่อให้ M1 เริ่มสร้าง engine กับ server ได้โดยไม่ต้องเดา

**Architecture:** โปรเจกต์สร้างจาก `create-rr-stack` แบบ prerender กับ SPA และ Tailwind type ของ model กับ op เป็น struct ของ Rust ที่ลงทะเบียนในสัญญา แล้วสร้างเป็น type ของ TypeScript ด้วย `bun run gen` sync client ฝั่งเครื่องอยู่ใน `web/app/sync/` และใน M0 มีเพียงตัวที่ทำงานในหน่วยความจำ ระบบจริงรันด้วย compose โดยมีตัวเชื่อมของ Cloudflare Tunnel แทน Caddy

**Tech Stack:** bun 1.4.5, Rust stable, Axum 0.8, sqlx 0.9, utoipa 6, Postgres 18, React Router 8, React 19, TanStack Query 5, Tailwind CSS 4, playwright-core, Docker, cloudflared

**Spec:** `docs/design/2026-11-26-issue-tracker-sync-engine-design.md` (ข้อ 15 หัวข้อ M0 คือเกณฑ์ของแผนนี้)

แผนนี้ครอบคลุม M0 เท่านั้น M1.1 ขึ้นไปมีแผนแยก

## Global Constraints

กติกาของ RR stack ซึ่งอยู่ใน `AGENTS.md` ของโปรเจกต์ ใช้กับทุก Task:

- งานเสร็จเมื่อ `bun run check` จบด้วย exit code 0 เท่านั้น ถ้าแก้อะไรใต้ `web/app` ต้องรัน `bun run e2e` ด้วย ถ้าแก้วิธีที่ server เสิร์ฟหน้าหรือไฟล์ ต้องรัน `bun run smoke` ด้วย
- ห้ามแก้ด้วยมือ: `openapi.json`, `web/app/api/**`, `server/.sqlx/**` ให้ใช้ `bun run gen` และ `cargo sqlx prepare -- --all-targets`
- type ที่ข้ามสายมาจาก struct ของ Rust ที่ derive `ToSchema` เท่านั้น ฝั่ง TypeScript import จาก `~/api` ห้ามเขียน type ที่ข้ามสายเอง
- ไม่มี `unwrap` หรือ `expect` นอกเทสต์ Rust เขียนแบบตรงไปตรงมา ไม่มี macro ของตัวเอง
- หน้าใหม่ต้องเพิ่มเส้นทางใน `web/e2e/pages.test.ts` flow ใหม่ต้องมีเทสต์ใน `web/e2e/` ที่หา element ด้วย role กับ label เท่านั้น
- script เป็น TypeScript ที่รันด้วย bun ห้ามเพิ่ม script ที่ใช้ได้เฉพาะ bash

ค่าจาก spec ที่ต้องตรงตัว:

- ชื่อฟิลด์ใน JSON เป็น snake_case ตามแบบของแม่แบบ
- สถานะ: `backlog`, `todo`, `in_progress`, `done`, `canceled`
- ความสำคัญ: `none`, `urgent`, `high`, `medium`, `low`
- id ของ issue, project และ label เป็น UUID v7 ที่เครื่องสร้าง id ของผู้ใช้และ workspace เป็นเลขที่ server ออก
- เวลาเป็นข้อความรูปแบบ RFC 3339
- key ของ workspace ตัวอย่างคือ `LMD` issue ที่ยังไม่มีเลขแสดงเป็น `LMD-…`

กติกาของ repo:

- Repo เป็น public: ห้ามมี secret, ชื่อโดเมนของระบบจริง, ข้อมูลของงานอื่น หรือข้อมูลส่วนตัวใน commit ใด
- commit `AGENTS.md` แต่ไม่ commit `CLAUDE.md`
- ห้ามใช้ชื่อ โลโก้ หน้าตา หรือโค้ดของ Linear
- `main` ถูกป้องกัน งานหนึ่ง Task คือหนึ่ง PR จาก branch ชื่อ `m0/<ชื่อสั้น>` การ merge ต้องให้เจ้าของสั่ง

## Review Focus

กรณีที่ spec บอกเป็นนัย และมีโอกาสทำให้ผู้ใช้เจอปัญหาสูงสุด แต่ละข้อมีเทสต์หรือขั้นตรวจใน Task ที่ระบุ

1. **ล้างค่ากับไม่แตะค่า** การแก้ issue ต้องแยกได้ระหว่าง "เอาผู้รับผิดชอบออก" กับ "ไม่ได้แก้ผู้รับผิดชอบ" ถ้าสับสน การเปลี่ยนสถานะจะลบผู้รับผิดชอบทิ้ง (Task 2 Step 1 และ Task 3 Step 1)
2. **issue ที่ยังไม่มีเลข** ต้องแสดงในรายการเป็น `LMD-…` และไม่ทำให้หน้าพัง (Task 3 Step 1 และ Task 4 Step 5)
3. **sortKey เท่ากัน** ลำดับที่แสดงต้องคงที่ทุกครั้ง โดยตัดสินด้วย id (Task 4 Step 1)
4. **เปิดลิงก์ตรงเข้าหน้าของแอป** ทั้งในเครื่องและผ่าน tunnel ต้องได้หน้าแอป ไม่ใช่ 404 (Task 4 Step 5 และ Task 5 Step 5)
5. **แอปของแม่แบบเปิดให้ใครก็สมัครได้** ระหว่างทดลอง tunnel แอปนี้อยู่บนอินเทอร์เน็ตจริง จึงต้องปิดทันทีเมื่อทดลองเสร็จ (Task 5 Step 8)

## Phase 0: ข้อเท็จจริงที่ตรวจกับแม่แบบและเอกสารแล้ว

**รับใช้เป้าหมายหลักอย่างไร:** ทุกคำสั่ง ชื่อไฟล์ และรูปแบบโค้ดในแผนนี้มาจากการอ่าน `create-rr-stack` 0.7.2 และเอกสารของ Cloudflare เมื่อ 26 พ.ย. 2569 จึงไม่เสียเวลากับสิ่งที่ไม่มีอยู่จริง

### ใช้ได้

| เรื่อง | ข้อเท็จจริง | ที่มาในโปรเจกต์ที่สร้าง |
|---|---|---|
| สร้างโปรเจกต์ | `bun create rr-stack <ชื่อ> --spa --tailwind --yes --no-git --no-install` ไม่ยอมสร้างลงโฟลเดอร์ที่ไม่ว่าง ชื่อต้องเป็นตัวพิมพ์เล็ก | `create-rr-stack` |
| คำสั่ง | `bun run dev`, `check`, `smoke`, `e2e`, `gen` | `package.json`, `scripts/` |
| สิ่งที่ `check` ต้องมี | Postgres ที่รันอยู่, `DATABASE_URL` ใน `.env` และ sqlx-cli | `scripts/check.ts` |
| พอร์ตตอนพัฒนา | API ใช้ `PORT` ใน `.env` ค่าตั้งต้น 3000, Vite ใช้ 5173 | `.env.example`, `web/vite.config.ts` |
| ลงทะเบียน type ที่ไม่มีเส้นทางใดคืน | เพิ่มใน `components(schemas(...))` ของ `ExtraSchemas` | `server/src/lib.rs` |
| เทสต์ว่า type อยู่ในสัญญา | `serde_json::to_value(server::openapi())` แล้วดู `components.schemas` | `server/tests/events.rs` |
| enum ที่มี tag | `#[serde(tag = "type", rename_all = "...")]` กับ variant ที่มีฟิลด์ | `server/src/events.rs` |
| เวลาใน JSON | ฟิลด์ `String` ที่มี `#[schema(format = DateTime)]` ค่าจาก `to_rfc3339()` | `server/src/todos.rs` |
| เทสต์หน่วยฝั่งเว็บ | `bun test app` ใช้ `import { expect, test } from "bun:test"` | `web/package.json` |
| เส้นทางของเว็บ | `route("<path>", "routes/<file>.tsx")` ใน `web/app/routes.ts` | `web/app/routes.ts` |
| หน้าที่ไม่ให้ค้นเจอ | ใส่ `<meta name="robots" content="noindex" />` ใน JSX ของหน้า | `web/app/routes/playground-layout.tsx` |
| เทสต์ทุกหน้า | array `pages` ใน `web/e2e/pages.test.ts` ทุกหน้าต้องตอบ 200 และมี `h1` | `web/e2e/pages.test.ts` |
| ตัวช่วยของ e2e | `withPages(name, count, test)`, `settle(page)`, `unique(prefix)` | `web/e2e/browser.ts` |
| token ของธีม | บล็อก `@theme` ใน `web/app/app.css` มี `--color-ink`, `--color-paper`, `--color-patina`, `--color-signal`, `--color-fault` และฟอนต์ IBM Plex ที่มีอักษรไทย | `web/app/app.css` |
| compose สำหรับ production | ไฟล์ทับที่ใช้ `ports: !reset []` กับ `db` และ `app` และตั้ง `TRUST_PROXY: "true"` | `compose.prod.yml` |
| งานของ CI | `check` บน `ubuntu-latest` กับ `windows-latest` และ `docker` | `.github/workflows/ci.yml` |
| Cloudflare Tunnel | ต้องมีโดเมนในบัญชี Cloudflare ตัวเชื่อมรันด้วย `cloudflared tunnel --no-autoupdate run --token <token>` และต่อออกอย่างเดียว | developers.cloudflare.com/tunnel/setup/ |
| WebSocket ผ่าน Cloudflare | ใช้ได้ทุกแผน การเชื่อมต่อที่เงียบถูกปิดหลังเวลาที่เอกสารไม่ระบุ | developers.cloudflare.com/network/websockets/ |
| IP ของผู้ใช้ | Cloudflare ส่ง `CF-Connecting-IP` และต่อท้าย `X-Forwarded-For` | developers.cloudflare.com/fundamentals/reference/http-headers/ |

### ห้ามทำ

- แก้ `openapi.json` หรือไฟล์ใต้ `web/app/api/` เอง
- เขียน interface ของ `Issue`, `Op` หรือ type อื่นที่ข้ามสายใน TypeScript
- ใช้ extractor ของ axum ตรง ๆ แทน `AppJson`, `AppPath`, `AppQuery`
- หา element ในเทสต์ e2e ด้วย class ของ CSS
- เก็บ store หรือ sync client ไว้ในตัวแปรระดับ module
- ใช้ tunnel ชั่วคราว (`trycloudflare.com`) เป็นทางขึ้นระบบจริง
- ใส่ `restart: unless-stopped` ในไฟล์ compose ของ tunnel เพราะระบบจริงเปิดเฉพาะเวลาที่ใช้งาน

### ยังไม่มีเอกสารยืนยัน แผนนี้จึงมีขั้นตรวจเอง

| เรื่อง | ตรวจที่ไหน |
|---|---|
| utoipa สร้าง schema ของ `Option<T>` กับ enum ที่มี tag ซ้อนกันตามที่ต้องการ | Task 2 Step 4 และ Task 3 Step 3 ซึ่งใช้ type ที่สร้างได้จริง |
| `bun test` อ่าน alias `~/` จาก `web/tsconfig.json` | Task 3 Step 2 |
| ชื่อ check ที่ GitHub รายงานสำหรับงานของ CI | Task 1 Step 8 |
| image ของ cloudflared รุ่นที่ระบุมีอยู่จริง | Task 5 Step 3 |
| WebSocket ที่เงียบอยู่ได้นานแค่ไหนผ่าน tunnel และ IP ที่ server เห็นเป็นของผู้ใช้จริงหรือไม่ | Task 5 Step 6 และ Step 7 |

## โครงไฟล์

```
(สร้างจากแม่แบบ) AGENTS.md  package.json  compose.yml  compose.prod.yml  Dockerfile
                 scripts/  server/  web/  openapi.json  .github/workflows/ci.yml
server/src/sync.rs                    type ของ model, op และเหตุผลการปฏิเสธ       (เจ้าของ)
server/tests/sync_contract.rs         เทสต์ว่า type อยู่ในสัญญา                    (เจ้าของ)
web/app/sync/                         สัญญาของ sync client และตัวในหน่วยความจำ    (เจ้าของ)
web/app/components/sync-provider.tsx  ตัวเชื่อม sync client กับ React              (jxyluvcode)
web/app/lib/group-issues.ts           จัดกลุ่มและเรียง issue                       (jxyluvcode)
web/app/routes/issues.tsx             หน้ารายการ issue                            (jxyluvcode)
web/e2e/issues.test.ts                เทสต์เบราว์เซอร์ของหน้ารายการ                (jxyluvcode)
compose.tunnel.yml                    ระบบจริงหลัง Cloudflare Tunnel               (เจ้าของ)
docs/design/                          เอกสารออกแบบ, แผนงาน, mockup, บันทึกการทดลอง
```

ลำดับ: Task 1 → 2 → 3 → 4 ส่วน Task 5 และ Task 6 เริ่มได้ทันทีหลัง Task 1 และ Task 7 ทำท้ายสุด

---

### Task 1: โปรเจกต์ RR stack ใน repo

**รับใช้เป้าหมายหลักอย่างไร:** ทำให้ข้อ "รันโปรเจกต์ที่สร้างจาก RR stack ได้" และ "ทุก PR ถูกตรวจด้วยคำสั่งของ stack" เป็นจริงก่อนมีโค้ดของ Lamdap บรรทัดแรก

**ผู้ทำ:** เจ้าของ

**Files:**
- มีอยู่แล้ว: `README.md`, `LICENSE`, `.gitignore`, `docs/design/**`
- Create: ทุกไฟล์ที่ `create-rr-stack` สร้าง ยกเว้น `README.md` ส่วน `CLAUDE.md` อยู่ในเครื่องได้แต่ไม่ถูก commit
- Modify: `README.md`, `.gitignore` (แทนด้วยของแม่แบบ)

**Interfaces:**
- Consumes: ไม่มี
- Produces: คำสั่ง `bun run dev`, `check`, `smoke`, `e2e`, `gen` ที่ root, crate ของ server ที่มี lib ชื่อ `server`, alias `~/` ที่ชี้ไป `web/app/`, งานของ CI ชื่อ `check` กับ `docker`

- [ ] **Step 1: ตรวจเครื่องมือ**

Run: `bun --version && docker --version && cargo --version && cargo sqlx --version`
Expected: bun 1.4.0 ขึ้นไป และทุกคำสั่งพิมพ์เลขรุ่น ถ้า `cargo sqlx` ไม่มี ให้รัน `cargo install sqlx-cli --no-default-features --features postgres,rustls`

- [ ] **Step 2: สร้างโปรเจกต์ในโฟลเดอร์ชั่วคราว**

ตัวสร้างไม่ยอมสร้างลงโฟลเดอร์ที่ไม่ว่าง จึงสร้างข้างนอกแล้วคัดลอกเข้ามา

```bash
SCAFFOLD="$(mktemp -d)"
(cd "$SCAFFOLD" && bun create rr-stack lamdap --spa --tailwind --yes --no-git --no-install)
ls "$SCAFFOLD/lamdap"
```

Expected: เห็น `AGENTS.md`, `server`, `web`, `scripts`, `compose.yml`, `package.json` ใน `$SCAFFOLD/lamdap`

- [ ] **Step 3: คัดลอกเข้า repo**

จาก root ของ repo บน branch `m0/scaffold`:

```bash
cp "$SCAFFOLD/lamdap/README.md" /tmp/lamdap-generated-readme.md
rm "$SCAFFOLD/lamdap/README.md"
grep -qx 'CLAUDE.md' .git/info/exclude || echo 'CLAUDE.md' >> .git/info/exclude
cp -a "$SCAFFOLD/lamdap/." .
rm -rf "$SCAFFOLD"
git status --short | head -20
```

Expected: `.gitignore` ถูกแทนด้วยของแม่แบบ, `LICENSE` กับ `docs/design` ยังอยู่ และ `git status` ไม่แสดง `.env` หรือ `CLAUDE.md` เพราะบรรทัดที่เพิ่มใน `.git/info/exclude` ทำให้ git ของเครื่องนี้มองข้ามไฟล์นั้น

- [ ] **Step 4: เพิ่มวิธีพัฒนาใน `README.md`**

เพิ่มหัวข้อ `## Development` ก่อนหัวข้อ `## License` เนื้อหา: สิ่งที่ต้องติดตั้ง (Docker, Rust stable, bun 1.4 ขึ้นไป, sqlx-cli) และตารางคำสั่งห้าตัว โดยนำคำอธิบายของแต่ละคำสั่งจาก `/tmp/lamdap-generated-readme.md` พร้อมประโยคว่าโปรเจกต์สร้างจาก RR stack และลิงก์ไป `AGENTS.md` แล้วลบไฟล์ใน `/tmp`

- [ ] **Step 5: ติดตั้งและเปิดฐานข้อมูล**

Run: `(cd web && bun install --frozen-lockfile) && docker compose up -d --wait db`
Expected: จบด้วย exit code 0 และ `docker compose ps` แสดง `db` เป็น healthy

ถ้าพอร์ต 3000 ของเครื่องถูกใช้อยู่ ให้ตั้ง `PORT` ใน `.env` เป็นค่าอื่น ไฟล์นี้ไม่ถูก commit

- [ ] **Step 6: รันคำสั่งตรวจทั้งสามของ stack**

Run: `bun run check && bun run smoke && bun run e2e`
Expected: ทั้งสามคำสั่งจบด้วย exit code 0 ครั้งแรกใช้เวลาหลายนาทีเพราะ compile Rust

- [ ] **Step 7: Commit, push และเปิด PR**

```bash
git add -A
git status --short | grep -E 'CLAUDE.md|\.env$' && echo "หยุด: มีไฟล์ที่ห้าม commit"
git commit -m "chore: scaffold from RR stack"
git push -u origin m0/scaffold
gh pr create --fill
```

Expected: บรรทัด grep ไม่พิมพ์อะไร และ PR ถูกเปิด

- [ ] **Step 8: หลัง CI ผ่านและ merge แล้ว ให้ `main` บังคับใช้ check**

Run: `gh pr checks <เลข PR>` เพื่ออ่านชื่อ check ที่ GitHub รายงานจริง แล้วใส่ชื่อของงาน `check` ทั้งสองระบบปฏิบัติการและงาน `docker` ลงใน `contexts`:

```bash
gh api -X PUT repos/Useless007/lamdap/branches/main/protection --input - <<'JSON'
{
  "required_status_checks": { "strict": false, "contexts": ["check (ubuntu-latest)", "check (windows-latest)", "docker"] },
  "enforce_admins": false,
  "required_pull_request_reviews": { "required_approving_review_count": 1 },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
JSON
gh api repos/Useless007/lamdap/branches/main/protection --jq '.required_status_checks.contexts'
```

Expected: คำสั่งสุดท้ายพิมพ์ชื่อ check ทั้งสาม ถ้าชื่อที่ `gh pr checks` รายงานต่างจากตัวอย่าง ให้ใช้ชื่อที่รายงาน

---

### Task 2: type ของ model และ op ในสัญญา

**รับใช้เป้าหมายหลักอย่างไร:** สัญญาของ sync client ใน Task 3 ต้องใช้ type ของ `Issue` กับ `Op` กติกาของ stack ห้ามเขียน type เหล่านี้ใน TypeScript เอง Task นี้จึงทำให้มันมาจาก Rust ตั้งแต่วันแรก

**ผู้ทำ:** เจ้าของ โดย jxyluvcode ตรวจ PR เพราะเป็นสัญญาที่ทั้งสองฝั่งใช้

**Files:**
- Create: `server/src/sync.rs`
- Modify: `server/src/lib.rs` (ประกาศโมดูล และ `ExtraSchemas`)
- Test: `server/tests/sync_contract.rs`
- สร้างด้วยคำสั่ง: `openapi.json`, `web/app/api/**`

**Interfaces:**
- Consumes: `ExtraSchemas` และ `server::openapi()` จาก Task 1
- Produces: type ต่อไปนี้ใน `server::sync` และชื่อเดียวกันใน `~/api` หลังรัน `bun run gen`

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "snake_case")]
pub enum IssueStatus { Backlog, Todo, InProgress, Done, Canceled }

// derive และ rename_all แบบเดียวกัน
pub enum IssuePriority { None, Urgent, High, Medium, Low }
pub enum MemberRole { Owner, Member }
pub enum RejectReason {
    Invalid, Forbidden, NotFound, StaleDescription, WorkspaceFull,
    ServerError, IdReuse, Voided, IncompatibleVersion,
}

// struct ทุกตัว: #[derive(Clone, Debug, PartialEq, Serialize, Deserialize, ToSchema)]
// ฟิลด์เวลาทุกฟิลด์มี #[schema(format = DateTime)]
pub struct Workspace { pub id: i64, pub name: String, pub key: String }
pub struct Member { pub user_id: i64, pub display_name: String, pub role: MemberRole, pub removed: bool }
pub struct Project {
    pub id: String, pub name: String, pub description: String, pub color: String,
    pub sort_key: String, pub deleted_at: Option<String>,
}
pub struct Label { pub id: String, pub name: String, pub color: String, pub deleted_at: Option<String> }
pub struct Issue {
    pub id: String, pub number: Option<i64>, pub title: String,
    pub status: IssueStatus, pub priority: IssuePriority,
    pub assignee_id: Option<i64>, pub project_id: Option<String>, pub sort_key: String,
    pub created_by: i64, pub created_at: String, pub updated_at: String, pub deleted_at: Option<String>,
}
pub struct IssueDescription { pub issue_id: String, pub text: String, pub rev: i64 }
pub struct IssueLabel { pub issue_id: String, pub label_id: String, pub removed: bool }

/// มีฟิลด์นี้ = เปลี่ยนผู้รับผิดชอบ, `user_id` เป็น null = เอาออก
pub struct AssigneeChange { pub user_id: Option<i64> }
/// มีฟิลด์นี้ = เปลี่ยน project, `project_id` เป็น null = เอาออก
pub struct ProjectChange { pub project_id: Option<String> }

// enum ที่มี tag ทั้งสองตัวใช้ derive ชุดเดียวกับ struct
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum DescriptionBase { Rev { rev: i64 }, AfterMutation { uid: String } }

#[serde(tag = "type", rename_all = "snake_case")]
pub enum Op {
    CreateIssue {
        id: String, title: String, sort_key: String,
        status: Option<IssueStatus>, priority: Option<IssuePriority>,
        assignee_id: Option<i64>, project_id: Option<String>, description: Option<String>,
    },
    UpdateIssue {
        id: String, title: Option<String>, status: Option<IssueStatus>, priority: Option<IssuePriority>,
        assignee: Option<AssigneeChange>, project: Option<ProjectChange>,
        sort_key: Option<String>, deleted: Option<bool>,
    },
    SaveDescription { issue_id: String, text: String, base: DescriptionBase },
    CreateProject { id: String, name: String, sort_key: String, description: Option<String>, color: Option<String> },
    UpdateProject {
        id: String, name: Option<String>, description: Option<String>, color: Option<String>,
        sort_key: Option<String>, deleted: Option<bool>,
    },
    CreateLabel { id: String, name: String, color: Option<String> },
    UpdateLabel { id: String, name: Option<String>, color: Option<String>, deleted: Option<bool> },
    SetIssueLabel { issue_id: String, label_id: String, removed: bool },
    UpdateWorkspace { name: Option<String>, key: Option<String> },
}
```

ขอบเขตของ Task นี้คือรูปร่างของข้อมูลเท่านั้น การตรวจความยาว รูปแบบของ id และกติกาอื่นเป็นงานของ M1.1 ซึ่งจะเปลี่ยนฟิลด์เป็น newtype โดยไม่เปลี่ยนรูปร่างใน JSON รูปร่างชุดนี้เป็นฉบับแรก M1.1 แก้ได้เมื่อทั้งสองคนเห็นชอบ

- [ ] **Step 1: เขียนเทสต์ที่ยังไม่ผ่าน**

`server/tests/sync_contract.rs`:

```rust
#![allow(clippy::unwrap_used)]

use serde_json::{Value, json};
use server::sync::{DescriptionBase, Op};

fn schemas() -> Value {
    serde_json::to_value(server::openapi()).unwrap()["components"]["schemas"].clone()
}

#[test]
fn sync_types_are_in_the_contract() {
    let schemas = schemas();
    for name in [
        "Workspace", "Member", "Project", "Label", "Issue", "IssueDescription", "IssueLabel",
        "IssueStatus", "IssuePriority", "MemberRole", "RejectReason",
        "AssigneeChange", "ProjectChange", "DescriptionBase", "Op",
    ] {
        assert!(schemas[name].is_object(), "{name} schema missing");
    }
}

#[test]
fn enum_values_match_the_design() {
    let schemas = schemas();
    assert_eq!(schemas["IssueStatus"]["enum"], json!(["backlog", "todo", "in_progress", "done", "canceled"]));
    assert_eq!(schemas["IssuePriority"]["enum"], json!(["none", "urgent", "high", "medium", "low"]));
}

#[test]
fn an_update_tells_clearing_from_not_touching() {
    let clear: Op = serde_json::from_value(json!({"type": "update_issue", "id": "a", "assignee": {"user_id": null}})).unwrap();
    let Op::UpdateIssue { assignee, status, .. } = clear else { panic!("wrong variant") };
    assert_eq!(assignee.unwrap().user_id, None);
    assert_eq!(status, None);

    let keep: Op = serde_json::from_value(json!({"type": "update_issue", "id": "a", "status": "done"})).unwrap();
    let Op::UpdateIssue { assignee, .. } = keep else { panic!("wrong variant") };
    assert!(assignee.is_none());
}

#[test]
fn ops_are_tagged_json() {
    let op: Op = serde_json::from_value(json!({"type": "set_issue_label", "issue_id": "i", "label_id": "l", "removed": false})).unwrap();
    assert_eq!(serde_json::to_value(&op).unwrap()["type"], "set_issue_label");

    let save: Op = serde_json::from_value(json!({
        "type": "save_description", "issue_id": "i", "text": "ข้อความ",
        "base": {"kind": "after_mutation", "uid": "u1"}
    })).unwrap();
    let Op::SaveDescription { base, .. } = save else { panic!("wrong variant") };
    assert_eq!(base, DescriptionBase::AfterMutation { uid: "u1".to_owned() });
    assert!(serde_json::from_value::<Op>(json!({"type": "delete_issue", "id": "a"})).is_err());
}
```

- [ ] **Step 2: รันเทสต์ให้เห็นว่าไม่ผ่าน**

Run: `(cd server && cargo test --test sync_contract)`
Expected: compile ไม่ผ่าน เพราะยังไม่มี `server::sync`

- [ ] **Step 3: เขียน `server/src/sync.rs` และลงทะเบียน**

เขียน type ตามบล็อก Interfaces ใน `server/src/lib.rs` เพิ่ม `pub mod sync;` และเพิ่ม type ทั้ง 15 ตัวใน `components(schemas(...))` ของ `ExtraSchemas` ต่อจาก `events::TodoEvent`

- [ ] **Step 4: รันเทสต์ให้ผ่าน แล้วสร้างสัญญาใหม่**

Run: `(cd server && cargo test --test sync_contract) && bun run gen && git status --short`
Expected: PASS 4 เทสต์ และ `git status` แสดงว่า `openapi.json` กับไฟล์ใต้ `web/app/api/` เปลี่ยน

ตรวจว่า type ที่สร้างได้ใช้งานได้: เปิด `web/app/api/types.gen.ts` แล้วยืนยันว่า `Op` เป็น union ที่แยกด้วยฟิลด์ `type` และ `assignee` ใน variant ของ `update_issue` เป็นฟิลด์ที่ไม่บังคับ ถ้าไม่เป็นตามนี้ ให้หยุดและรายงาน เพราะ Task 3 พึ่งรูปร่างนี้

- [ ] **Step 5: รันคำสั่งตรวจของ stack**

Run: `bun run check`
Expected: exit code 0

- [ ] **Step 6: Commit**

```bash
git add server/src/sync.rs server/src/lib.rs server/tests/sync_contract.rs openapi.json web/app/api
git commit -m "feat(sync): put model and op types in the contract"
```

---

### Task 3: สัญญาของ sync client และตัวที่ทำงานในหน่วยความจำ

**รับใช้เป้าหมายหลักอย่างไร:** ทำให้ข้อ "ทำงานบนสัญญาของ sync client ทั้งที่ยังไม่มี engine จริง" เป็นจริง jxyluvcode จึงสร้างหน้าจอของ M1.3 ได้ตั้งแต่จบ M0 โดยไม่รอ M1.1

**ผู้ทำ:** เจ้าของ

**Files:**
- Create: `web/app/sync/client.ts`, `ids.ts`, `memory-client.ts`, `sample.ts`, `index.ts`
- Modify: `web/package.json` และ `web/bun.lock` (เพิ่ม `fractional-indexing`)
- Test: `web/app/sync/ids.test.ts`, `web/app/sync/memory-client.test.ts`

**Interfaces:**
- Consumes: type `Workspace`, `Member`, `Project`, `Label`, `Issue`, `IssueDescription`, `IssueLabel`, `Op`, `DescriptionBase`, `RejectReason` จาก `~/api`
- Produces (ทั้งหมด export จาก `~/sync`):

```ts
export interface Snapshot {
  readonly workspace: Workspace;
  readonly members: ReadonlyMap<number, Member>;            // key: user_id
  readonly projects: ReadonlyMap<string, Project>;
  readonly labels: ReadonlyMap<string, Label>;
  readonly issues: ReadonlyMap<string, Issue>;
  readonly descriptions: ReadonlyMap<string, IssueDescription>;  // key: issue_id
  readonly issueLabels: ReadonlyMap<string, IssueLabel>;         // key: issueLabelKey()
}
export type Connection = "connecting" | "online" | "offline" | "server-unavailable";
export interface SyncStatus {
  readonly ready: boolean;
  readonly connection: Connection;
  readonly pendingCount: number;
  readonly paused: boolean;          // แท็บอื่นเป็นตัวทำงาน (M3)
  readonly updateRequired: boolean;  // server ใช้ protocol รุ่นใหม่กว่า
  readonly halted: boolean;          // sync ผิดพลาดและหยุดส่ง
}
export interface Failure {
  readonly id: string;
  readonly reason: RejectReason;
  readonly ops: readonly Op[];
  readonly createdAt: string;
  readonly conflict?: { readonly issueId: string; readonly mine: string; readonly latest: string; readonly latestRev: number };
}
export interface SyncClient {
  subscribe(listener: () => void): () => void;
  getSnapshot(): Snapshot;
  getStatus(): SyncStatus;
  getFailures(): readonly Failure[];
  mutate(ops: readonly Op[]): void;
  descriptionBase(issueId: string): DescriptionBase;
  retryFailure(id: string): void;
  discardFailure(id: string): void;
  close(): void;
}
export interface SyncClientOptions {
  readonly workspaceId: number;
  readonly userId: number;
  readonly deviceKey?: string;       // ค่าตั้งต้น "main"
}
export interface SeedData {
  workspace: Workspace; members: Member[]; projects: Project[]; labels: Label[];
  issues: Issue[]; descriptions: IssueDescription[]; issueLabels: IssueLabel[];
}
export interface MemoryDev {
  setStatus(patch: Partial<SyncStatus>): void;
  addFailure(failure: Pick<Failure, "reason" | "ops" | "conflict">): void;
}
export function createMemorySyncClient(
  options: SyncClientOptions & { seed: SeedData; now?: () => Date },
): SyncClient & { readonly dev: MemoryDev };
export function sampleSeed(now: Date): SeedData;

// ids.ts
export function newId(now?: number): string;                              // UUID v7 ตัวพิมพ์เล็ก
export function issueLabelKey(issueId: string, labelId: string): string;  // `${issueId}:${labelId}`
export function sortKeyBetween(before: string | null, after: string | null): string;
export function compareBySortKey(a: { sort_key: string; id: string }, b: { sort_key: string; id: string }): number;
export function formatIssueIdentifier(key: string, number: number | null | undefined): string;  // "LMD-12" หรือ "LMD-…"
```

ข้อผูกพันของ `SyncClient` ทุกตัว ซึ่ง engine จริงใน M1.1 ต้องรักษาเช่นกัน:

- `getSnapshot`, `getStatus`, `getFailures` คืนค่าตัวเดิม (เทียบด้วย `===`) จนกว่าจะมีการเปลี่ยนแปลง
- เมื่อข้อมูลเปลี่ยน Map ของ model ที่ไม่ถูกแตะ และแถวที่ไม่ถูกแตะ ยังเป็น object ตัวเดิม
- op ทั้งชุดของ `mutate` หนึ่งครั้งมีผลพร้อมกัน และ listener ถูกเรียกหนึ่งครั้ง
- `mutate` ไม่โยน error จากเรื่องเครือข่ายหรือการปฏิเสธ ผลเหล่านั้นไปปรากฏที่ `getFailures`

พฤติกรรมของตัวในหน่วยความจำ ซึ่งเลียนแบบ server เท่าที่หน้าจอต้องใช้:

- สถานะเริ่มต้นคือ `{ ready: true, connection: "online", pendingCount: 0, paused: false, updateRequired: false, halted: false }`
- `create_issue`: `number` เป็นเลขมากสุดที่มีอยู่บวกหนึ่ง, `created_at` กับ `updated_at` เป็น `now().toISOString()`, `created_by` เป็น `userId`, ค่าที่ไม่ส่งมาใช้ `status: "backlog"`, `priority: "none"`, `assignee_id: null`, `project_id: null`, `deleted_at: null` และสร้างแถวของรายละเอียดด้วย `text` จาก `description` หรือข้อความว่าง กับ `rev: 0`
- `update_issue`: แก้เฉพาะฟิลด์ที่ส่งมา, `assignee` กับ `project` ที่ส่งมาตั้งค่าตาม `user_id` กับ `project_id` รวมถึง `null`, `deleted: true` ตั้ง `deleted_at` เป็นเวลาปัจจุบัน และ `deleted: false` ตั้งเป็น `null`, ตั้ง `updated_at` ทุกครั้ง
- `save_description`: ถ้า `base` เป็น `rev` ที่ตรงกับค่าปัจจุบัน ให้บันทึกและเพิ่ม `rev` หนึ่ง ถ้าไม่ตรง ไม่แก้อะไร และเพิ่ม failure เหตุผล `stale_description` พร้อม `conflict` ส่วน `base` แบบ `after_mutation` ให้บันทึกเสมอ
- `descriptionBase(issueId)` คืน `{ kind: "rev", rev: <rev ปัจจุบัน> }`
- `set_issue_label`: สร้างหรือแก้แถวเดียวที่ key จาก `issueLabelKey`
- `create_project`, `update_project`, `create_label`, `update_label`, `update_workspace`: สร้างหรือแก้แบบเดียวกัน ค่าตั้งต้นของ `color` คือ `"#6b7280"` และของ `description` คือข้อความว่าง
- op ที่อ้างถึงแถวที่ไม่มีอยู่: ทั้งชุดไม่มีผล และเพิ่ม failure เหตุผล `not_found`
- สร้างด้วย id ที่มีอยู่แล้ว: ทั้งชุดไม่มีผล และเพิ่ม failure เหตุผล `invalid`
- `retryFailure` ลบ failure แล้วเรียก `mutate` ด้วย op เดิม, `discardFailure` ลบอย่างเดียว, `close` ทำให้ listener ไม่ถูกเรียกอีก

ข้อมูลตัวอย่างจาก `sampleSeed`: workspace `{ id: 1, name: "Lamdap", key: "LMD" }`, สมาชิก 2 คน (`Mali` เป็น owner มี `user_id: 1`, `Tawan` เป็น member มี `user_id: 2`), project 2 อัน, label 3 อัน, issue 12 รายการที่มีทั้งหัวข้อไทยและอังกฤษกระจายครบ 5 สถานะ ในนั้นต้องมีรายการหัวข้อ `แก้หน้า login` ที่มีเลข, รายการหัวข้อ `เขียน README` ที่ `number` เป็น `null` และหนึ่งรายการที่ `deleted_at` ไม่เป็น `null` ทุก issue มีแถวของรายละเอียด

- [ ] **Step 1: เขียนเทสต์ที่ยังไม่ผ่าน**

`web/app/sync/ids.test.ts`:

```ts
import { expect, test } from "bun:test";
import { compareBySortKey, formatIssueIdentifier, issueLabelKey, newId, sortKeyBetween } from "./ids";

test("newId makes lowercase UUID v7 that sorts by time", () => {
  const a = newId(1_000);
  const b = newId(2_000);
  expect(a).toMatch(/^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
  expect(a < b).toBe(true);
});

test("sort keys land between their neighbours and ties break by id", () => {
  const first = sortKeyBetween(null, null);
  const last = sortKeyBetween(first, null);
  const middle = sortKeyBetween(first, last);
  expect(first < middle && middle < last).toBe(true);
  const rows = [{ sort_key: "a0", id: "b" }, { sort_key: "a0", id: "a" }, { sort_key: "Zz", id: "c" }];
  expect(rows.sort(compareBySortKey).map((r) => r.id)).toEqual(["c", "a", "b"]);
});

test("identifiers and label keys", () => {
  expect(formatIssueIdentifier("LMD", 12)).toBe("LMD-12");
  expect(formatIssueIdentifier("LMD", null)).toBe("LMD-…");
  expect(issueLabelKey("i", "l")).toBe("i:l");
});
```

`web/app/sync/memory-client.test.ts`:

```ts
import { expect, mock, test } from "bun:test";
import type { Op } from "~/api";
import { createMemorySyncClient, issueLabelKey, newId, sampleSeed } from "./index";

const setup = () => {
  const seed = sampleSeed(new Date("2026-01-01T00:00:00.000Z"));
  const client = createMemorySyncClient({ workspaceId: 1, userId: 1, seed, now: () => new Date("2026-02-02T00:00:00.000Z") });
  return { seed, client };
};
const NOW = "2026-02-02T00:00:00.000Z";
const createIssue = (title: string): Extract<Op, { type: "create_issue" }> => ({ type: "create_issue", id: newId(), title, sort_key: "a0" });

test("the seed has an issue without a number and one in the trash", () => {
  const issues = [...setup().client.getSnapshot().issues.values()];
  expect(issues).toHaveLength(12);
  expect(issues.some((i) => i.title === "เขียน README" && i.number == null)).toBe(true);
  expect(issues.some((i) => i.title === "แก้หน้า login" && i.number != null)).toBe(true);
  expect(issues.some((i) => i.deleted_at != null)).toBe(true);
});

test("creating an issue fills in what the server would", () => {
  const { seed, client } = setup();
  const max = Math.max(...seed.issues.map((i) => i.number ?? 0));
  const op = createIssue("งานใหม่");
  client.mutate([op]);
  expect(client.getSnapshot().issues.get(op.id)).toEqual({
    id: op.id, number: max + 1, title: "งานใหม่", status: "backlog", priority: "none",
    assignee_id: null, project_id: null, sort_key: "a0",
    created_by: 1, created_at: NOW, updated_at: NOW, deleted_at: null,
  });
  expect(client.getSnapshot().descriptions.get(op.id)).toEqual({ issue_id: op.id, text: "", rev: 0 });
});

test("an update changes only what it names, and can clear the assignee", () => {
  const { seed, client } = setup();
  const target = seed.issues.find((i) => i.assignee_id != null)!;
  client.mutate([{ type: "update_issue", id: target.id, status: "done" }]);
  expect(client.getSnapshot().issues.get(target.id)).toMatchObject({ status: "done", assignee_id: target.assignee_id, updated_at: NOW });
  client.mutate([{ type: "update_issue", id: target.id, assignee: { user_id: null } }]);
  expect(client.getSnapshot().issues.get(target.id)).toMatchObject({ status: "done", assignee_id: null });
});

test("whatever a mutation did not touch keeps its identity", () => {
  const { seed, client } = setup();
  const before = client.getSnapshot();
  expect(client.getSnapshot()).toBe(before);
  client.mutate([{ type: "update_issue", id: seed.issues[0]!.id, priority: "high" }]);
  const after = client.getSnapshot();
  expect(after).not.toBe(before);
  expect(after.projects).toBe(before.projects);
  expect(after.issues.get(seed.issues[1]!.id)).toBe(before.issues.get(seed.issues[1]!.id));
});

test("one mutation notifies once", () => {
  const { client } = setup();
  const listener = mock(() => {});
  const stop = client.subscribe(listener);
  client.mutate([createIssue("หนึ่ง"), createIssue("สอง")]);
  expect(listener).toHaveBeenCalledTimes(1);
  stop();
  client.mutate([createIssue("สาม")]);
  expect(listener).toHaveBeenCalledTimes(1);
});

test("the trash and back", () => {
  const { seed, client } = setup();
  const id = seed.issues.find((i) => i.deleted_at == null)!.id;
  client.mutate([{ type: "update_issue", id, deleted: true }]);
  expect(client.getSnapshot().issues.get(id)!.deleted_at).toBe(NOW);
  client.mutate([{ type: "update_issue", id, deleted: false }]);
  expect(client.getSnapshot().issues.get(id)!.deleted_at).toBeNull();
});

test("a description is saved only on top of the latest revision", () => {
  const { seed, client } = setup();
  const id = seed.issues[0]!.id;
  const first = client.descriptionBase(id);
  expect(first).toEqual({ kind: "rev", rev: client.getSnapshot().descriptions.get(id)!.rev });
  client.mutate([{ type: "save_description", issue_id: id, text: "ฉบับแรก", base: first }]);
  const saved = client.getSnapshot().descriptions.get(id)!;
  expect(saved.text).toBe("ฉบับแรก");
  client.mutate([{ type: "save_description", issue_id: id, text: "ฉบับที่ทับ", base: first }]);
  expect(client.getSnapshot().descriptions.get(id)!.text).toBe("ฉบับแรก");
  expect(client.getFailures()).toHaveLength(1);
  expect(client.getFailures()[0]).toMatchObject({
    reason: "stale_description",
    conflict: { issueId: id, mine: "ฉบับที่ทับ", latest: "ฉบับแรก", latestRev: saved.rev },
  });
});

test("a batch that points at nothing changes nothing and is reported", () => {
  const { client } = setup();
  const before = client.getSnapshot();
  client.mutate([createIssue("ok"), { type: "update_issue", id: newId(), status: "done" }]);
  expect(client.getSnapshot()).toBe(before);
  expect(client.getFailures()[0]!.reason).toBe("not_found");
});

test("attaching a label twice leaves one row", () => {
  const { seed, client } = setup();
  const issue_id = seed.issues[0]!.id;
  const label_id = seed.labels[0]!.id;
  const attach = { type: "set_issue_label", issue_id, label_id, removed: false } satisfies Op;
  client.mutate([attach]);
  client.mutate([{ ...attach, removed: true }]);
  client.mutate([attach]);
  expect(client.getSnapshot().issueLabels.get(issueLabelKey(issue_id, label_id))).toEqual({ issue_id, label_id, removed: false });
  expect(client.getFailures()).toHaveLength(0);
});

test("failures can be retried or discarded, and the dev handle sets status", () => {
  const { client } = setup();
  client.dev.addFailure({ reason: "server_error", ops: [createIssue("ลองใหม่")] });
  client.retryFailure(client.getFailures()[0]!.id);
  expect(client.getFailures()).toHaveLength(0);
  expect([...client.getSnapshot().issues.values()].some((i) => i.title === "ลองใหม่")).toBe(true);
  client.dev.addFailure({ reason: "forbidden", ops: [createIssue("ทิ้ง")] });
  client.discardFailure(client.getFailures()[0]!.id);
  expect(client.getFailures()).toHaveLength(0);
  client.dev.setStatus({ connection: "offline", pendingCount: 3 });
  expect(client.getStatus()).toMatchObject({ connection: "offline", pendingCount: 3, ready: true, halted: false });
});
```

- [ ] **Step 2: รันเทสต์ให้เห็นว่าไม่ผ่าน**

Run: `(cd web && bun add fractional-indexing@4.0.0 && bun test app/sync)`
Expected: FAIL เพราะยังไม่มี `./ids` และ `./index` ถ้าข้อผิดพลาดเป็นการหา `~/api` ไม่เจอ แปลว่า `bun test` ไม่อ่าน alias ให้หยุดและรายงาน

- [ ] **Step 3: เขียน `client.ts`, `ids.ts`, `sample.ts`, `memory-client.ts`, `index.ts`**

- `client.ts` มีเฉพาะ interface กับ type ตามบล็อก Interfaces และ import type จาก `~/api`
- `ids.ts` เขียน UUID v7 เอง: 48 บิตแรกเป็นเวลาหน่วยมิลลิวินาที ที่เหลือมาจาก `crypto.getRandomValues` แล้วตั้งบิตของ version กับ variant ส่วน `sortKeyBetween` เรียก `generateKeyBetween` จาก `fractional-indexing` และ `compareBySortKey` เทียบข้อความด้วยตัวดำเนินการ `<` ไม่ใช้ `localeCompare`
- `memory-client.ts` คำนวณทั้งชุดบนสำเนาของ Map ที่ถูกแตะก่อน แล้วสลับ snapshot ครั้งเดียว เพื่อให้ชุดที่ล้มเหลวไม่ทิ้งร่องรอย
- ถ้า type ที่สร้างจากสัญญาใช้กับเทสต์ข้างบนไม่ได้ ให้แก้ที่ Rust ใน `server/src/sync.rs` แล้วรัน `bun run gen` ห้ามเขียน type ทับใน TypeScript

- [ ] **Step 4: รันเทสต์ให้ผ่าน**

Run: `(cd web && bun test app/sync)`
Expected: PASS 13 เทสต์

- [ ] **Step 5: รันคำสั่งตรวจของ stack**

Run: `bun run check && bun run e2e`
Expected: exit code 0 ทั้งคู่

- [ ] **Step 6: Commit**

```bash
git add web/app/sync web/package.json web/bun.lock
git commit -m "feat(sync): add the sync client contract and an in-memory client"
```

---

### Task 4: หน้ารายการ issue หน้าแรก

**รับใช้เป้าหมายหลักอย่างไร:** เป็นหน้าจอแรกที่ใช้สัญญาจาก Task 3 จริง และเป็นจุดเริ่มที่ jxyluvcode รับช่วงต่อ

**ผู้ทำ:** เจ้าของวางโครง jxyluvcode ตรวจ PR และรับช่วงดูแลไฟล์ชุดนี้หลัง merge

**Files:**
- Create: `web/app/lib/group-issues.ts`, `web/app/components/sync-provider.tsx`, `web/app/routes/issues.tsx`
- Modify: `web/app/routes.ts`, `web/e2e/pages.test.ts`
- Test: `web/app/lib/group-issues.test.ts`, `web/e2e/issues.test.ts`

**Interfaces:**
- Consumes: ทุกอย่างที่ Task 3 ส่งออกจาก `~/sync` และ type `Issue`, `IssueStatus` จาก `~/api`
- Produces:

```ts
// web/app/lib/group-issues.ts
export const STATUS_ORDER: readonly IssueStatus[];            // backlog, todo, in_progress, done, canceled
export const STATUS_LABELS: Record<IssueStatus, string>;      // Backlog, Todo, In progress, Done, Canceled
export function groupIssuesByStatus(issues: Iterable<Issue>): ReadonlyArray<{ status: IssueStatus; issues: Issue[] }>;

// web/app/components/sync-provider.tsx
export function SyncProvider(props: { client: SyncClient; children: ReactNode }): ReactNode;
export function useSyncClient(): SyncClient;                               // โยน Error เมื่อไม่มี SyncProvider ครอบ
export function useSnapshot<T>(selector: (snapshot: Snapshot) => T): T;    // selector ต้องเป็นฟังก์ชันตัวเดิมทุกรอบ render
export function useSyncStatus(): SyncStatus;
export function useFailures(): readonly Failure[];
```

`groupIssuesByStatus` คืน 5 กลุ่มเสมอตามลำดับของ `STATUS_ORDER`, ตัดรายการที่ `deleted_at` ไม่เป็น `null` และเรียงในกลุ่มด้วย `compareBySortKey`

หน้า `/issues`:

- อยู่ใน `site-layout` ก่อนเส้นทาง `*` ใน `web/app/routes.ts`
- มี `<meta name="robots" content="noindex" />` และ `<h1>Issues</h1>`
- สร้าง sync client ด้วย `useState(() => createMemorySyncClient({ workspaceId: 1, userId: 1, seed: sampleSeed(new Date()) }))` แล้วส่งผ่าน `SyncProvider`
- แสดงสถานะการเชื่อมต่อจาก `useSyncStatus()`
- มีช่องกรอกที่มี `aria-label="New issue title"` เมื่อกด Enter และข้อความไม่ว่าง ให้เรียก `mutate` ด้วย `create_issue` ที่ `id` มาจาก `newId()` และ `sort_key` มาจาก `sortKeyBetween(<sort_key มากสุดของกลุ่ม backlog หรือ null>, null)` แล้วล้างช่อง
- แต่ละกลุ่มเป็น `<section aria-label={STATUS_LABELS[status]}>` ที่มีหัวข้อ `h2` เป็นชื่อกลุ่ม แต่ละรายการแสดง `formatIssueIdentifier(workspace.key, issue.number)` และหัวข้อ

- [ ] **Step 1: เขียนเทสต์ของ `groupIssuesByStatus` ที่ยังไม่ผ่าน**

`web/app/lib/group-issues.test.ts`:

```ts
import { expect, test } from "bun:test";
import type { Issue } from "~/api";
import { groupIssuesByStatus } from "./group-issues";

const issue = (id: string, patch: Partial<Issue>): Issue => ({
  id, number: 1, title: id, status: "todo", priority: "none", assignee_id: null, project_id: null,
  sort_key: "a0", created_by: 1, created_at: "2026-01-01T00:00:00.000Z", updated_at: "2026-01-01T00:00:00.000Z",
  deleted_at: null, ...patch,
});

test("five groups in status order, no trashed issues, ties broken by id", () => {
  const groups = groupIssuesByStatus([
    issue("b", {}), issue("a", {}), issue("z", { sort_key: "Zz" }),
    issue("gone", { deleted_at: "2026-01-02T00:00:00.000Z" }), issue("d", { status: "done" }),
  ]);
  expect(groups.map((g) => g.status)).toEqual(["backlog", "todo", "in_progress", "done", "canceled"]);
  expect(groups[1]!.issues.map((i) => i.id)).toEqual(["z", "a", "b"]);
  expect(groups[3]!.issues.map((i) => i.id)).toEqual(["d"]);
  expect(groups[0]!.issues).toEqual([]);
});
```

Run: `(cd web && bun test app/lib/group-issues.test.ts)`
Expected: FAIL เพราะยังไม่มี `./group-issues`

- [ ] **Step 2: เขียน `group-issues.ts` แล้วรันเทสต์ให้ผ่าน**

Run: `(cd web && bun test app/lib/group-issues.test.ts)`
Expected: PASS 1 เทสต์

- [ ] **Step 3: เขียน `sync-provider.tsx`**

`useSnapshot` ใช้ `useSyncExternalStore(client.subscribe, client.getSnapshot, client.getSnapshot)` แล้วคำนวณ `selector(snapshot)` ใน `useMemo` ที่ขึ้นกับ `snapshot` และ `selector` ส่วน `useSyncStatus` กับ `useFailures` ใช้ `useSyncExternalStore` กับ `getStatus` และ `getFailures` ตรง ๆ ห้ามเก็บ client ไว้ในตัวแปรระดับ module

- [ ] **Step 4: เขียน `routes/issues.tsx` และเพิ่มเส้นทาง**

เพิ่ม `route("issues", "routes/issues.tsx"),` ใน `web/app/routes.ts` ต่อจากบล็อกของ `playground-layout` และก่อนเส้นทาง `*`

- [ ] **Step 5: เขียนเทสต์เบราว์เซอร์**

ใน `web/e2e/pages.test.ts` เพิ่ม `"/issues"` ต่อท้าย array `pages`

`web/e2e/issues.test.ts`:

```ts
import { expect, test } from "bun:test";
import { settle, unique, withPages } from "./browser";

test("the issue list shows seeded issues, including one without a number", () =>
  withPages("issues seeded", 1, async ([page]) => {
    await page!.goto("/issues");
    await settle(page!);
    await page!.getByRole("heading", { name: "Issues", level: 1 }).waitFor();
    await page!.getByText("แก้หน้า login").waitFor();
    await page!.getByText("เขียน README").waitFor();
    expect(await page!.getByText("LMD-…").count()).toBeGreaterThan(0);
  }));

test("an issue is created from the keyboard and lands in Backlog", () =>
  withPages("issues create", 1, async ([page]) => {
    const title = unique("งานใหม่จากเทสต์");
    await page!.goto("/issues");
    await settle(page!);
    const input = page!.getByLabel("New issue title");
    await input.fill(title);
    await input.press("Enter");
    await page!.getByRole("region", { name: "Backlog" }).getByText(title).waitFor();
    expect(await input.inputValue()).toBe("");
  }));
```

- [ ] **Step 6: รันคำสั่งตรวจของ stack**

Run: `bun run check && bun run e2e`
Expected: exit code 0 ทั้งคู่ และผลของ `e2e` มีเทสต์ของ `/issues` ผ่าน

- [ ] **Step 7: Commit**

```bash
git add web/app/lib web/app/components/sync-provider.tsx web/app/routes/issues.tsx web/app/routes.ts web/e2e
git commit -m "feat(web): add the first issue list on the sync client contract"
```

---

### Task 5: ทางขึ้นระบบจริงผ่าน Cloudflare Tunnel

**รับใช้เป้าหมายหลักอย่างไร:** ทำให้ข้อ "แอปเปิดได้จากอินเทอร์เน็ตผ่าน Cloudflare Tunnel จากเครื่องของเจ้าของ" เป็นจริง และได้ตัวเลขสองตัวที่ M1.1 ต้องใช้ คือเวลาที่ WebSocket ที่เงียบอยู่ได้ และวิธีที่ server เห็น IP ของผู้ใช้

**ผู้ทำ:** เจ้าของ มีขั้นที่เจ้าของต้องลงมือเองเพราะเป็นการสร้าง secret

**Files:**
- Create: `compose.tunnel.yml`, `docs/design/m0-tunnel-notes.md`
- Modify: `.env.example` (เพิ่มตัวแปรที่ถูก comment ไว้), `DEPLOY.md` (เพิ่มหัวข้อ ถ้าโปรเจกต์ที่สร้างไม่มีไฟล์นี้ ให้สร้างใหม่)

**Interfaces:**
- Consumes: `compose.yml`, `Dockerfile` และแอปของแม่แบบจาก Task 1
- Produces: คำสั่งเปิดและปิดระบบหลัง tunnel และบันทึกผลการทดลอง

- [ ] **Step 1: เขียน `compose.tunnel.yml`**

```yaml
# The app on your own machine, reachable through Cloudflare Tunnel. No port is published.
#   docker compose -f compose.yml -f compose.tunnel.yml up -d --build --wait
#   docker compose -f compose.yml -f compose.tunnel.yml down
# Needs SITE_URL, POSTGRES_PASSWORD and TUNNEL_TOKEN in .env.
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env (openssl rand -hex 24)}
    ports: !reset []

  app:
    build:
      args:
        SITE_URL: ${SITE_URL:?Set SITE_URL in .env, for example https://app.example.com}
    environment:
      # cloudflared is the one proxy in front: trust its X-Forwarded-For for the sign-in rate limit.
      TRUST_PROXY: "true"
    ports: !reset []

  tunnel:
    image: cloudflare/cloudflared:2026.8.3
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN:?Set TUNNEL_TOKEN in .env}
    depends_on:
      - app
```

ใน `.env.example` เพิ่มท้ายไฟล์:

```
# Running behind Cloudflare Tunnel (compose.tunnel.yml): the tunnel's token, and a real database password.
# TUNNEL_TOKEN=
# POSTGRES_PASSWORD=
```

- [ ] **Step 2 [เจ้าของทำเอง]: สร้าง tunnel และตั้งค่าในเครื่อง**

1. ที่ dashboard ของ Cloudflare ไปที่ Networking > Tunnels สร้าง tunnel แล้วคัดลอก token
2. ใน tunnel นั้นเพิ่มเส้นทางแบบ Published application: เลือกชื่อย่อยกับโดเมน และตั้ง Service URL เป็น `http://app:3000`
3. ใส่ค่าใน `.env` ของเครื่อง ซึ่งไม่ถูก commit: `TUNNEL_TOKEN`, `POSTGRES_PASSWORD` (สร้างด้วย `openssl rand -hex 24`) และ `SITE_URL` เป็น `https://<ชื่อที่เลือก>`

Expected: `grep -c -E '^(TUNNEL_TOKEN|POSTGRES_PASSWORD|SITE_URL)=' .env` พิมพ์ `3` และ `git status --short` ไม่แสดง `.env`

- [ ] **Step 3: เปิดระบบ**

Run: `docker compose -f compose.yml -f compose.tunnel.yml up -d --build --wait`
Expected: ทุก service ขึ้น และ dashboard ของ Cloudflare แสดง tunnel เป็น Healthy ถ้า image รุ่นที่ระบุดึงไม่ได้ ให้ใช้รุ่นล่าสุดที่มี tag เป็นเลขรุ่น แล้วแก้ไฟล์ให้ตรง

- [ ] **Step 4: ตรวจว่าไม่มีพอร์ตถูกเปิดบนเครื่อง**

Run: `docker compose -f compose.yml -f compose.tunnel.yml ps --format '{{.Service}} {{.Ports}}'`
Expected: ไม่มีบรรทัดใดมี `0.0.0.0` หรือ `127.0.0.1`

- [ ] **Step 5: ตรวจจากอินเทอร์เน็ต**

แทน `<host>` ด้วยชื่อที่เลือก:

```bash
UA='Mozilla/5.0 (X11; Linux x86_64) lamdap-check'
curl -s -A "$UA" -o /dev/null -w '%{http_code} %{content_type}\n' https://<host>/api/health
curl -s -A "$UA" -o /dev/null -w '%{http_code} %{content_type}\n' https://<host>/issues
```

Expected: `200 application/json` และ `200 text/html...` จากนั้นทั้งสองคนเปิด `https://<host>/app/todos` จากเครือข่ายของตัวเอง สมัครบัญชีคนละบัญชี คนหนึ่งเพิ่ม todo แล้วอีกคนเห็นโดยไม่ต้อง reload

- [ ] **Step 6: วัดเวลาที่ WebSocket ที่เงียบอยู่ได้**

```bash
bun -e 'const t = Date.now(); const ws = new WebSocket("wss://<host>/api/events"); ws.onopen = () => console.log("open"); ws.onclose = (e) => { console.log("closed after", Math.round((Date.now() - t) / 1000), "s, code", e.code); process.exit(0); }; setTimeout(() => { console.log("still open after 900 s"); process.exit(0); }, 900_000);'
```

Expected: พิมพ์จำนวนวินาทีที่การเชื่อมต่อถูกปิด หรือ `still open after 900 s` ห้ามมีการแก้ไขข้อมูลในแอประหว่างวัด เพราะจะมีข้อความวิ่งผ่านการเชื่อมต่อ

- [ ] **Step 7: ตรวจ IP ที่ server เห็น**

จากเครือข่ายของเจ้าของ ส่งการ login ที่รหัสผ่านผิด 11 ครั้งภายในหนึ่งนาที:

```bash
for i in $(seq 1 11); do curl -s -A 'Mozilla/5.0 (X11; Linux x86_64) lamdap-check' -o /dev/null -w '%{http_code} ' -X POST https://<host>/api/auth/login -H 'content-type: application/json' -d '{"email":"nobody@example.com","password":"wrong-password"}'; done; echo
```

Expected: สิบครั้งแรกได้ `401` และครั้งที่สิบเอ็ดได้ `429` จากนั้นให้ jxyluvcode ส่งคำสั่งเดียวกันหนึ่งครั้งจากเครือข่ายของตัวเองภายในนาทีเดียวกัน ถ้าได้ `401` แปลว่า server นับแยกตาม IP ของผู้ใช้จริง ถ้าได้ `429` แปลว่า server เห็นทุกคนเป็น IP เดียว ถ้าได้ `403` จาก Cloudflare เอง แปลว่าคำขอถูกกันก่อนถึงแอป ให้ทำจากเบราว์เซอร์แทน ให้บันทึกผลตามจริง

- [ ] **Step 8: ปิดระบบทันที**

Run: `docker compose -f compose.yml -f compose.tunnel.yml down`
Expected: ทุก service หยุด และ `https://<host>/api/health` ไม่ตอบ 200 อีก ข้อมูลยังอยู่ใน volume

แอปของแม่แบบเปิดให้ใครก็สมัครได้ จึงห้ามเปิดทิ้งไว้ จนกว่า M1.2 จะปิดการสมัคร

- [ ] **Step 9: บันทึกผลและเอกสาร**

`docs/design/m0-tunnel-notes.md` มีวันที่, รุ่นของ cloudflared, ผลของ Step 4 ถึง 7 และข้อเสนอสำหรับ M1.1 ว่าสัญญาณชีพควรถี่แค่ไหน และควรอ่าน IP จาก header ใด เอกสารนี้ห้ามมีชื่อโดเมนจริง

ใน `DEPLOY.md` เพิ่มหัวข้อ "On your own machine behind Cloudflare Tunnel" ที่มีคำสั่งเปิดกับปิด, ตัวแปรสามตัวใน `.env` และคำเตือนว่าไฟล์นี้ไม่มี `restart` เพราะตั้งใจให้เปิดเฉพาะเวลาที่ใช้งาน

Run: `bun run check`
Expected: exit code 0

```bash
git add compose.tunnel.yml .env.example DEPLOY.md docs/design/m0-tunnel-notes.md
git commit -m "feat(deploy): run behind Cloudflare Tunnel on your own machine"
```

---

### Task 6: เลือกหน้าตา

**รับใช้เป้าหมายหลักอย่างไร:** ทำให้ข้อ "เจ้าของเลือกหน้าตาแล้ว" เป็นจริง งานหน้าจอใน M1.3 และ M2 จึงมีทิศทางเดียวและไม่เหมือน Linear

**ผู้ทำ:** jxyluvcode กับเจ้าของ โดยเจ้าของเป็นผู้ตัดสิน

**Files:**
- Create: `docs/design/mockups/a/index.html`, `docs/design/mockups/b/index.html`, `docs/design/mockups/c/index.html`
- Create: `docs/design/visual-direction.md`
- Modify: `web/app/app.css`

**Interfaces:**
- Consumes: โครงหน้าจอในข้อ 11 ของ spec และข้อมูลตัวอย่างชุดเดียวกับ `sampleSeed`
- Produces: ค่าใหม่ของ token ใน `@theme` ของ `web/app/app.css` โดยคงชื่อ token ของแม่แบบไว้ (`--color-ink`, `--color-ink-soft`, `--color-paper`, `--color-patina`, `--color-patina-deep`, `--color-signal`, `--color-fault`, `--color-fault-deep`, `--font-sans`, `--font-mono`) เพราะหน้าของแม่แบบยังใช้อยู่จนถึง M1.2 และเพิ่ม token ของ 5 สถานะชื่อ `--color-status-backlog`, `--color-status-todo`, `--color-status-in-progress`, `--color-status-done`, `--color-status-canceled`

- [ ] **Step 1: ทำ mockup สามทิศทาง**

แต่ละทิศทางเป็นไฟล์ HTML กับ CSS ไฟล์เดียวที่เปิดได้โดยไม่ต้อง build และแสดงสามหน้าจอ: รายการ issue, หน้า issue และ command palette ที่เปิดอยู่ ทุกหน้าจอต้องมี:

- หัวข้อทั้งไทยและอังกฤษ และฟอนต์ที่มีอักษรไทย
- ครบ 5 สถานะและ 5 ระดับความสำคัญ
- รายการที่ยังไม่มีเลข, ตัวบอกว่า offline และมี 3 รายการรอส่ง

สามทิศทางต้องต่างกันจริงในโทนสี, ตัวอักษร และความหนาแน่น และไม่มีทิศทางใดใช้ชุดสี ไอคอน หรือการจัดวางที่ทำให้นึกถึง Linear

- [ ] **Step 2: ตรวจความต่างสี**

ตรวจคู่สีตัวหนังสือกับพื้นของแต่ละทิศทาง ทั้งโหมดสว่างและโหมดมืดถ้ามี
Expected: ตัวหนังสือปกติมีอัตราส่วนอย่างน้อย 4.5:1 และตัวหนังสือขนาดใหญ่อย่างน้อย 3:1 ตามเกณฑ์ WCAG AA

- [ ] **Step 3 [เจ้าของตัดสิน]: เลือกหนึ่งทิศทาง**

เจ้าของกับ jxyluvcode เปิดทั้งสามไฟล์ในเบราว์เซอร์ เจ้าของเลือกหนึ่งทิศทาง และบอกสิ่งที่อยากปรับ
Expected: มีคำตัดสินเป็นลายลักษณ์อักษรใน PR

- [ ] **Step 4: บันทึกผลและใส่ token**

`docs/design/visual-direction.md` ระบุทิศทางที่เลือก เหตุผล และตาราง token แล้วใส่ค่าชุดเดียวกันใน `@theme` ของ `web/app/app.css`

Run: `bun run check && bun run e2e`
Expected: exit code 0 ทั้งคู่

```bash
git add docs/design/mockups docs/design/visual-direction.md web/app/app.css
git commit -m "docs: choose the visual direction and set theme tokens"
```

---

### Task 7: ปิด M0

**รับใช้เป้าหมายหลักอย่างไร:** ตรวจเป้าหมายหลักทีละข้อกับหลักฐาน

**ผู้ทำ:** เจ้าของ

**Files:**
- Create: `docs/design/rr-stack-feedback.md`

- [ ] **Step 1 [jxyluvcode ทำเอง]: ยืนยันว่ารันในเครื่องของตัวเองได้**

clone repo แล้วทำตามหัวข้อ Development ใน `README.md`
Expected: `bun run dev` ขึ้น, `http://localhost:5173/issues` แสดงรายการ issue และยืนยันเป็นความเห็นใน PR ของ Task นี้

- [ ] **Step 2: เริ่มบันทึกสิ่งที่จะนำกลับไปปรับปรุง RR stack**

`docs/design/rr-stack-feedback.md` เริ่มจากรายการในข้อ 13 ของ spec แล้วเพิ่มทุกอย่างที่ติดขัดระหว่างทำ Task 1 ถึง 6 แต่ละข้อมีสิ่งที่เจอ ไฟล์ที่เกี่ยวข้อง และข้อเสนอ

- [ ] **Step 3: ตรวจเกณฑ์ปิด M0**

กรอกตารางในหัวข้อถัดไปด้วยลิงก์ของหลักฐาน แล้ว commit

## เกณฑ์ปิด M0

เกณฑ์มาจากข้อ 15 ของ spec ทุกแถวต้องมีหลักฐานก่อนถือว่า M0 เสร็จ

| เกณฑ์ | หลักฐาน | สถานะ |
|---|---|---|
| โปรเจกต์จาก RR stack อยู่ใน repo และ `check`, `smoke`, `e2e` ผ่านทั้งในเครื่องและใน CI | PR ของ Task 1 ที่ check ทั้งสามเป็นสีเขียว | |
| เปิด PR แล้ว CI ผ่าน | PR ของ Task 2 ถึง 4 | |
| มีสัญญาของ sync client พร้อมตัวในหน่วยความจำ และหน้ารายการ issue ที่ใช้สัญญานั้น | เทสต์ของ Task 3 และ Task 4 ผ่านใน CI | |
| เจ้าของเลือกหน้าตาแล้ว | `docs/design/visual-direction.md` | |
| ทั้งสองคนรัน `bun run dev` ในเครื่องของตัวเองได้ | ความเห็นยืนยันของ jxyluvcode ใน Task 7 Step 1 | |
| ทั้งสองคนเปิดลิงก์ของ tunnel ได้ และการอัปเดตสดทำงานผ่าน tunnel | `docs/design/m0-tunnel-notes.md` | |
| รู้ว่า WebSocket ที่เงียบอยู่ได้นานแค่ไหน และ IP ที่ server เห็นเป็นของผู้ใช้จริงหรือไม่ | เอกสารเดียวกัน | |

ตรวจซ้ำก่อนปิด:

- `git ls-files | grep -E '(^|/)CLAUDE\.md$|(^|/)\.env$'` ต้องไม่พบอะไร
- `git grep -n -E 'TUNNEL_TOKEN=.+|POSTGRES_PASSWORD=[0-9a-f]{16}'` ต้องไม่พบอะไร
- `git grep -n -iE 'interface (Issue|Project|Label|Member|Workspace)\b' -- web/app` ต้องไม่พบอะไร เพราะ type เหล่านี้ต้องมาจาก `~/api`
- `bun run check && bun run smoke && bun run e2e` จบด้วย exit code 0 บน `main`
- ระบบหลัง tunnel ปิดอยู่

## ไม่ทำใน M0

ตารางฐานข้อมูลของ Lamdap · push, pull, bootstrap และ poke · engine ฝั่งเครื่องตัวจริง · ตัวจำลอง · การตรวจข้อมูลของ op · ระบบปิดและรหัสเชิญ · การถอดหน้าตัวอย่างของแม่แบบ · คำสั่งเปิดปิดระบบที่สำรองข้อมูล · เดโมบน Vercel · IndexedDB · service worker · หน้าจออื่นนอกจากรายการ issue
