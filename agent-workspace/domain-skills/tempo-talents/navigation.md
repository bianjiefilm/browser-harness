# Tempo Talents (huixue-yuanban school platform)

Internal school SPA at `http://100.74.141.3:3000` (Tailscale-only, Vue3 + Ant
Design Vue, hash router). Used for driving real teacher/student flows against
the school's live server for documentation/verification tasks.

## Login

- Route: `/#/login`
- Fields: `input[placeholder="用户名/手机号"]`, `input[placeholder="密码"]`,
  submit button text `登 录` (note the space between the two characters).
- Session = `huixue_token` cookie, a JWT with a short-ish `exp`. Also mirrored
  (non-authoritatively) into `localStorage.huixue_user` /
  `localStorage.userInfo` as a plain JSON blob — these can go stale/empty
  independently of the cookie, don't trust them alone to prove login state.
- To check whether a session is actually valid (not just "UI looks logged
  in"), decode the JWT payload from the cookie and compare `exp` to now:
  ```js
  document.cookie.match(/huixue_token=([^;]+)/)[1]
  ```
  then base64url-decode the middle segment. A UI that shows a cached
  logged-in shell (avatar/name in header) while API calls silently 401
  ("未授权，请重新登录" toast) means the JWT expired but the SPA didn't force
  a redirect to `/login` — don't take the rendered header at face value.
- Known accounts (see project's own memory, not this skill): `teacher1` /
  `teacher123`, `student1` / `student1`-style creds, `school_admin` /
  `tempo123`. Ask the operator if unsure — do not guess.

## Classroom → 课程实践 → 查看成绩 flow (teacher role)

1. After login lands on `/#/classroom` ("我的课堂"), classroom cards render
   under `.classroom-card` (Ant Design `ant-card`), with a clickable cover at
   `.classroom-cover-gradient` (`cursor: pointer`, no native `<a href>` —
   navigation happens via an onClick handler, so a synthetic mouse click that
   isn't a *real* click event won't trigger it — see gotcha below).
2. Clicking a card navigates to `/#/classroom/<id>` (classroom detail). The
   left sidebar's "课程实践" tab is selected by default and lists chapters
   (`.ant-collapse-header`, e.g. "第一章 未分类") that must be expanded
   (click the header) to reveal individual course/关卡 rows.
3. Each course row has a "查看成绩" link/button (plain text match works:
   `[...document.querySelectorAll("button, a, span")].filter(e =>
   e.textContent.trim() === "查看成绩")`). Clicking it navigates to:
   ```
   /#/classroom/<classroomId>/course/<courseId>/grades
     ?courseName=<urlencoded name>&courseType=practice&practiceId=<id>
   ```
   `courseType=practice` is how you confirm you're looking at the
   `course-grades.vue` **practice**-type branch specifically (as opposed to
   BI/experiment-type courses which use a different courseType).
4. The grades page renders a real per-student table (student name, 学号,
   作业状态, 提交时间, 完成关卡 `x/N`, 关卡得分, 当前成绩) — when data is
   real you'll see differing per-student values (e.g. one student `0/12`
   score `0`, another `1/12` score `8.3`). If instead every row is
   identically zero/empty across all students, that's the known
   "all-zero" regression, not a real state — worth flagging, not silently
   working around.

## Gotcha: `click_at_xy` can silently land on the wrong browser tab

This school's real Chrome window tends to accumulate multiple duplicate
`http://100.74.141.3:3000/#/classroom` tabs left open from prior agent runs
(plus unrelated tabs like Vercel/GitHub/Google search — this is the
operator's real, shared Chrome, not a throwaway browser).

The daemon keeps one "current session" used by *both* `js()`/`page_info()`
and `click_at_xy()` (via `Input.dispatchMouseEvent`). If any CDP call in
between hits a "stale session" error, the daemon's fallback silently
re-attaches to `Target.getTargets()[0]` of the *real-page* subset — which is
whatever tab is first in Chrome's internal target list (often one of the old
duplicate tabs, not the one you just created/logged into). Symptoms:

- `page_info()` / `js("document.title")` keep reporting the tab you expect
  (title/URL look right), **but**:
- `click_at_xy(x, y)` produces *no* `mousedown`/`mouseup`/`click` DOM events
  at all on that page (verify by temporarily attaching listeners:
  `document.addEventListener("click", e => ..., true)` before clicking).
- Or worse: the click silently lands on a *different* stale tab (e.g. an old
  tab logged in as a different role/account) and navigates *that* tab, while
  your `page_info()` calls right after still show the tab you meant to be on
  until you query it again and discover it never moved.

Workaround that reliably worked here: skip `click_at_xy` for this app and
dispatch a **real DOM click from inside the page's own JS context** instead,
via `js(...)`. Since `js()` and `click_at_xy()` share the same session
selection logic, if `js()` is confirmed on the right tab (e.g.
`localStorage.getItem("huixue_user")` shows the expected username/role),
a same-context `element.click()` call is guaranteed to fire on that same
tab:

```js
[...document.querySelectorAll(".classroom-card")]
  .find(c => c.textContent.includes("Python程序设计实训班"))
  .querySelector(".classroom-cover-gradient")
  .click()
```

Before trusting *any* tab identity in this environment, verify with a
content check tied to the account (not just title/URL, which is identical
across the duplicate tabs):
```js
localStorage.getItem("huixue_user")   // {"username":"teacher1","role":"teacher",...}
```
`current_tab()` from the core helpers was observed returning a bogus
detached target (empty url/title) throughout this session — don't rely on it
here; use `page_info()` + the localStorage check above instead.

## Known UI bug (reproduced, not fixed here)

Cold hash-navigation + forced reload to a classroom detail deep link
(`goto_url("http://100.74.141.3:3000/#/classroom/<id>")` followed by
`js("location.reload(true)")`) renders a blank page: `#app` innerHTML stays
at `<!---->` (length 7) indefinitely — the SPA doesn't rehydrate correctly
from a cold hash-URL reload. Real UI-click navigation (login → click through
cards/tabs) does not hit this bug and is the reliable path.

## Related skill

For BI「项目实训」/「教学资源」and practice「查看挑战」/Monaco challenge routes (including window.open recovery), see `content-inventory.md`.
