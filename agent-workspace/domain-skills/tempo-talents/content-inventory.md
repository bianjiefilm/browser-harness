# Tempo Talents — Content inventory open paths

Durable map for opening BI trainings, coding challenges, and big-data
challenges during the 18-course content audit. Complements
`navigation.md` (login, classroom cards, grades).

Base: `http://100.74.141.3:3000` (Tailscale, Vue3 + AntDV, **hash router**).

## Hard rules

1. **Prefer `js()` + `element.click()`** over `click_at_xy`. School Chrome has
   many duplicate tabs; coordinate clicks can land on a stale tab while
   `page_info()` still looks correct.
2. **Do not cold-reload classroom deep links.**
   `goto_url(.../#/classroom/<id>)` + hard reload leaves `#app` at `<!---->`.
   Use in-SPA navigation only (card click, `location.hash = "..."`, or Vue
   `$router.push`). Challenge routes (`/#/course/challenge/...`) tolerate
   hash assignment without a full reload; still avoid `location.reload(true)`.
3. **Session truth = cookie `huixue_token` JWT**, not `localStorage.huixue_user`
   (often stale after role switch). Decode middle segment for `sub` / `role` /
   `exp`.
4. **Admin has no top-nav「我的课堂」** after login (lands on
   `/#/admin/dashboard`). Force classroom list with
   `location.hash = "/classroom"` or `$router.push("/classroom")`.

## Login (recap)

| Item | Value |
|------|--------|
| Route | `/#/login` |
| Username | `input[placeholder="用户名/手机号"]` |
| Password | `input[placeholder="密码"]` |
| Submit | button text **`登 录`** (space between 登 and 录) |
| Success | teacher → `/#/classroom`; admin → `/#/admin/dashboard` |

Known roles for inventory: `school_admin` (admin), `teacher1` (teacher).
Credentials live in project env, never in this skill.

### Reliable Vue router helper

```js
function vuePush(path) {
  const app = document.querySelector("#app");
  const router = app?.__vue_app__?.config?.globalProperties?.$router;
  if (router) { router.push(path); return "router"; }
  location.hash = path.startsWith("/") ? path : "/" + path;
  return "hash";
}
```

Use when cover/`进入课堂` click does not change content (seen for admin on
the classroom list: title flickers to「课堂详情」but cards remain).

## Classroom entry

```js
// Teacher: cover click works
[...document.querySelectorAll(".classroom-card")]
  .find(c => c.textContent.includes("Python程序设计实训班"))
  .querySelector(".classroom-cover-gradient")
  .click()
// → /#/classroom/<classroomId>   title: 课堂详情
```

- Card: `.classroom-card` (Ant `ant-card`)
- Cover: `.classroom-cover-gradient` (no `<a href>`, onClick only)
- Fallback label: button **`进入课堂`** (`ant-btn-link`) inside the card
- Admin fallback: `vuePush("/classroom/1696")` etc.

### Classroom detail left sidebar (labels)

| Label | Default? | Hash suffix |
|-------|----------|-------------|
| 课程实践 | yes (practice) | (none or practices) |
| 项目实训 | BI / project trainings | `/trainings` |
| 教学资源 | modules + files | `/resources` |
| 课程考核 | | |
| 学情分析 | | |
| 课堂云盘 | | |

Click leaf text **exact match** (`textContent.trim() === "项目实训"`). URL
examples:

- `/#/classroom/1696` — overview + 课程实践
- `/#/classroom/1696/trainings` — 项目实训 list
- `/#/classroom/1696/resources` — 教学资源
- `/#/classroom/1696/training/71` — single training handbook (BI)

---

## Path A — BI / 项目实训 (school_admin)

Sample: **电商销售BI分析实验班** (`classroomId=1696`).

1. Login as admin → `vuePush("/classroom")`.
2. Open classroom (cover / `进入课堂` / `vuePush("/classroom/1696")`).
3. Sidebar → **项目实训** → hash `.../trainings`.
4. Row shows training name + type badge (e.g. `DATA_ANALYSIS`) + actions:
   - **`查看详情`** — `button.ant-btn-primary.ant-btn-sm` (handbook)
   - **查看成绩** / **作业列表** / **设置** (other inventory steps)
5. Click **查看详情** → same tab:
   ```
   /#/classroom/<classroomId>/training/<trainingId>
   ```
   Title: **实训详情**. No `window.open`.

### Handbook structure (stable classes)

- Container: `.training-detail-container`
- Handbook block: `.handbook-section` > `.handbook-card` > `.handbook-content` > **`.markdown-body`**
- Section title text: e.g. `📄 实训手册` + H-level titles inside markdown
- TOC on the right under `📑 目录章节`
- CTA: **`👁️ 查看实训`** (opens BI designer — separate from handbook audit)

### 教学资源 tab

- Hash: `/#/classroom/<id>/resources`
- Wrapper: `.teaching-resources` / `.resources-header`
- Empty state (common on fresh BI classes):
  - copy: `暂无教学资源模块` / `请先创建模块来组织您的教学资源`
  - buttons: **添加模块**, **创建第一个模块**
- Help text: teachers may upload PDF/DOC/DOCX/PPT/PPTX/MP4
- When modules exist: list/open first module item by visible name or first
  row action; structure is module-first, then files under each module.

---

## Path B — Coding practice / 查看挑战 (teacher)

Sample: **Python程序设计实训班** (`1704`) → **Spark编程基础（Python版）**.

1. Login teacher → already on `/#/classroom`.
2. Cover-click classroom → `/#/classroom/1704`.
3. Sidebar **课程实践** (default). Expand chapter:
   ```js
   [...document.querySelectorAll(".ant-collapse-header")]
     .find(h => h.textContent.includes("第一章") || h.textContent.includes("未分类"))
     .click()
   ```
4. Course rows: `.course-item` with:
   - `.course-number` (e.g. `1.`)
   - **`.course-name-link.clickable`** — open course detail
   - status pill `学习中`
   - teacher action **`查看成绩`** (`ant-btn-default ant-btn-sm`)
5. Click course name → `/#/classroom/1704/course/29` (title: **课程详情**).
6. Task list: `.tasks-card` / `.tasks-container`; levels are collapse items
   `第N关：...` with meta `通关: x | 未通关: y | 可获金币: z | 类型: 实践题`.
7. Expand 第1关 if needed; click **`查看挑战`**
   (`button.ant-btn-primary` inside `.task-actions`).

### Popup behavior (critical)

`查看挑战` calls:

```js
window.open("/#/course/challenge/<courseId>/<taskId>", "_blank")
```

Harness/CDP often **blocks** the new tab. Pattern that works:

```js
// before click
window.__opened = [];
const _open = window.open;
window.open = function(...args) {
  window.__opened.push(args.map(String));
  try { return _open.apply(this, args); } catch (e) { return null; }
};

// after click
const raw = window.__opened[0][0]; // "/#/course/challenge/3/26"
const path = raw.replace(/^\/#/, "").replace(/^#/, "");
location.hash = path.startsWith("/") ? path : "/" + path;
// → /#/course/challenge/3/26
```

Verified Spark 第1关: **`/#/course/challenge/3/26`**  
(`courseId=3`, `taskId=26`, route name `ChallengeDetail`).

### Challenge page markers

| Marker | Notes |
|--------|--------|
| Title | `挑战详情 - Tempo Talents` |
| Header | 上一关 / title / 下一关 |
| Tabs | **过关任务**, **参考答案** |
| Handbook | markdown body (e.g. `…学习手册`) |
| Editor | **Monaco** (`.monaco-editor`, `.view-lines`) |
| Actions | 查看输入输出示例, 查看测试用例, **提交评测**, 重置全部/本页代码, **评 测** |
| Failure copy | `实训不存在或已删除`, `未授权` (toast/body) — stop and report |

---

## Path C — Big-data practice (teacher)

Sample: **大数据技术实训班** (`1705`) → **关卡1-Hadoop概述与集群搭建**.

Same as Path B with naming quirks:

- Course display name is already `关卡1-Hadoop概述与集群搭建` (prefix 关卡N-).
- Course detail: `/#/classroom/1705/course/42`
- Single task collapse: `第1关：Hadoop 概述与集群搭建`
- `查看挑战` → `window.open("/#/course/challenge/12/130", "_blank")`
- Same-tab recovery: `location.hash = "/course/challenge/12/130"`
- Challenge page: Monaco + markdown, tabs 过关任务 / 参考答案

---

## URL pattern cheat sheet

| Flow | Pattern |
|------|---------|
| Login | `/#/login` |
| My classrooms | `/#/classroom` |
| Classroom detail | `/#/classroom/<classroomId>` |
| Project trainings | `/#/classroom/<classroomId>/trainings` |
| Training handbook | `/#/classroom/<classroomId>/training/<trainingId>` |
| Teaching resources | `/#/classroom/<classroomId>/resources` |
| Practice course | `/#/classroom/<classroomId>/course/<courseId>` |
| Practice grades | `/#/classroom/<cId>/course/<courseId>/grades?courseType=practice&...` |
| Challenge | `/#/course/challenge/<catalogCourseId>/<taskId>` |

Note: challenge `courseId` is the **catalog/course content id**, not the
classroom course row id (e.g. classroom course `29` → challenge course `3`).

## Selectors that work

```text
.classroom-card
.classroom-cover-gradient
.ant-collapse-header
.course-item / .course-name-link.clickable
button text exact: 进入课堂 | 查看详情 | 查看成绩 | 查看挑战 | 登 录
.training-detail-container .handbook-content .markdown-body
.tasks-card .task-actions button.ant-btn-primary   (查看挑战)
.monaco-editor / .view-lines
.markdown-body
.teaching-resources
```

## Selectors / tactics that do **not** work reliably

- `click_at_xy` on this school Chrome (stale tab drift)
- Trusting `localStorage.huixue_user` for role after multi-account runs
- `current_tab()` helper (observed detached empty target)
- Cold `goto_url` + reload into `/#/classroom/<id>` (SPA blank)
- Assuming `查看挑战` stays in the current tab

## Failure strings to surface in inventory reports

- `实训不存在或已删除`
- `未授权` / `未授权，请重新登录`
- Empty teaching resources: `暂无教学资源模块` (not a hard error — record as empty)
- Blank `#app` after reload → navigation bug, retry via UI clicks only
