# راهنمای انتقال بخش ایجنت به سیستم دیگر

> هدف: بردن حلقهٔ ایجنت همراه با **کنترل مهارت‌ها (Skills)** و **ابزارها (Tools)** به یک سیستم/میزبان دیگر، با حداقل وابستگی به Gateway، کانال‌ها و UI.
> نوشته‌شده بر اساس ساختار واقعی مخزن `on4net/openclaw`.

---

## ۱. خلاصهٔ معماری: این سیستم از قبل دو نیمه دارد

کد از قبل به «هستهٔ قابل حمل» و «میزبان» تقسیم شده؛ درگاه (seam) رسمی آن `AgentHarnessHostCapabilities` است:

| نیمه | محل | وضعیت برای انتقال |
|---|---|---|
| هستهٔ حلقهٔ مدل | `packages/agent-core`، `packages/ai`، `packages/llm-core`، `packages/tool-call-repair` | تقریباً مستقل؛ فقط چند وابستگی workspace |
| قرارداد افزونه/هارنس | `packages/plugin-sdk`، `packages/model-catalog-core` | قابل حمل با نسخه‌بندی |
| میزبان ایجنت | `src/agents` (به‌ویژه `harness/`، `embedded-agent-runner/`، `agent-tools.ts`) | باید «برش» بخورد |
| مهارت‌ها | `src/skills` | نیمی عمومی، نیمی میزبان‌محور |
| نشست/حافظه/ترنسکریپت | `src/sessions`، `src/state`، `src/storage` (SQLite) | باید جایگزین شود |
| کانال/گیت‌وی/UI | `src/gateway`، `src/channels`، `src/auto-reply`، `ui/` | باید حذف شود |

### پکیج‌های مستقل موجود در `packages/`

| پکیج | نقش |
|---|---|
| `@openclaw/agent-core` | حلقهٔ مدل، ابزار، compaction، repetition guard، harness types |
| `@openclaw/ai` | آداپترهای provider و streaming (anthropic/openai/google/...) |
| `@openclaw/llm-core` | قراردادهای پیام/مدل سطح پایین |
| `@openclaw/model-catalog-core` | مرجع provider/model |
| `@openclaw/plugin-sdk` | مرز عمومی پلاگین‌ها (هارنس، provider، channel، tool) |
| `@openclaw/tool-call-repair` | ترمیم tool call خراب provider |
| `@openclaw/normalization-core` | نرمال‌سازی مشترک |

---

## ۲. مهارت‌ها (Skills) — چه چیزی جدا شود

قرارداد داده و تولید پرامپت **عمومی** است؛ کشف، مسیرها و اولویت‌ها **میزبان‌محور** هستند.

### ببر (قابل حمل)

| مورد | فایل |
|---|---|
| قرارداد `SkillSnapshot` (prompt + skills + filter/overrides) | `src/skills/types.ts` |
| تولید متن «available skills» با بودجهٔ توکن | `src/skills/loading/skill-contract.ts`، `src/skills/loading/skill-prompt-limits.ts` |
| مدل‌های `Skill` / `SkillEntry` / frontmatter / `requires.bins` | `src/skills/types.ts` |
| اعتبارسنجی و امنیت مهارت | `src/skills/security` |

### جایگزین کن (میزبان‌محور)

- **ترتیب اولویت ۷ منبع** (workspace → `.agents/skills` → `~/.agents/skills` → state-dir → workshop → bundled → extraDirs/plugin) —
  مستند: `docs/tools/skills.md`. در سیستم جدید باید ریشه‌های خودت را تعریف کنی.
- **کشف skills** با worker: `src/skills/runtime/discovery.worker.ts` و `src/skills/discovery` → درگاه `SkillCatalog` خودت.
- **کانال‌های مهارت** (workshop / library / remote / ClawHub / node-hosted) → اختیاری؛ فقط اگر لازم داری.
- **گرهٔ اتصال فعلی** که باید عوض شود: `src/agents/harness/host-skills.ts`
  (`bindHostSkillCatalog` + `prepareHostSkillLibraryResources`) — از state-dir، sandbox و permission-root استفاده می‌کند.
- **کنترل دسترسی مهارت** (allowlist هر ایجنت + override هر نشست) روی `skillFilter` / `skillOverrides` در snapshot —
  مستند: `docs/tools/skills-config.md`.

### حداقل قرارداد قابل حمل

```ts
type SkillSnapshot = {
  prompt: string;                       // متن آمادهٔ تزریق در system prompt
  skills: Array<{ name: string; source?: { filePath: string } }>;
  skillFilter?: string[];               // فیلتر سطح ایجنت
  skillOverrides?: Record<string, boolean>;
  promptFormatVersion?: number;
};

type SkillCatalog = {
  loadSnapshot(input: { agentId: string; workspaceDir: string }): Promise<SkillSnapshot>;
  readInstructions(filePath: string, signal?: AbortSignal): Promise<string>;
};
```

---

## ۳. ابزارها (Tools) — چه چیزی جدا شود

### ببر (قرارداد + سیاست)

| مورد | فایل |
|---|---|
| قرارداد `AnyAgentTool` (`schema` + `execute(toolCallId, params, signal, onUpdate)`) | `src/agents/tools/common.ts` |
| سیاست ابزار (allow/deny، نرمال‌سازی نام، متادیتای پلاگین) | `src/agents/tool-policy.ts` + `tool-policy-*.ts` |
| اعمال سیاست روی ابزارهای هر تلاش | `src/agents/harness/prompt-tool-policy.ts` (`toolsAllow`، tool-search / code-mode) |
| دروازهٔ اجرای ابزار | `src/agents/agent-tool-availability.ts` (`gateAgentToolExecution`) |
| کاتالوگ ابزار هر تلاش | `src/agents/embedded-agent-runner/run/attempt-tool-catalog.ts` |
| قلاب `before_tool_call` + تأییدها | `src/agents/agent-tools.before-tool-call.ts` و فایل‌های `.policy` / `.approval` / `.wrapper` |

### جدا کن یا حذف کن (میزبان‌محور)

- `src/agents/agent-tools.ts` (~۳۴KB) به `auto-reply`، `gateway`، `plugins`، `process supervisor`، `config` وابسته است؛
  این فایل باید به یک `ToolRegistry` ساده شکسته شود.
- **ابزارهایی که در سیستم مقصد معنا ندارند** (حذف):
  - `message-tool*`، `sessions-*`، `gateway-tool`، `nodes-tool`
  - `computer-tool*`، `screen-tool`، `presence-tool`، `portal-tool`
  - `cron-tool*`، `dashboard-tool`، `secrets-tool`، `transcripts-tool`
  - `mobile-ui-tool`، `theme-tool`، `subagents-tool`، `system-agent-tool`، `skill-workshop-tool`
- **ابزارهای عمومی که می‌بری ولی مجوزشان را خودت بده**:
  `read`/`write`/`edit`، `apply-patch`، `terminal-tool` (exec)، `web-fetch`، `web-search`، `pdf-tool`، `ask-user-tool`.
- **منابع ابزار که باید port شوند**: ابزارهای پلاگین (`src/plugins` → `tool-*`) و MCP (`src/mcp`).
- **محدودیت‌های امنیتی ابزار**: sandbox / elevated / permission-mode —
  مستند: `docs/gateway/config-tools/tool-policy.md`. تصمیم دربارهٔ آن‌ها مال میزبان جدید است.

---

## ۴. هفت درگاهی که باید در سیستم مقصد بنویسی

موتور ایجنت همه‌چیز را از این‌ها می‌گیرد؛ با تعریف این‌ها بقیهٔ وابستگی‌ها قطع می‌شوند:

```ts
type Ports = {
  host: AgentHarnessHostCapabilities;   // assertActive, bindToolSurface, runBeforeToolCall,
                                        // requestApproval/waitForApproval, bindModelExecution,
                                        // preparedEnvironment, trajectory
  skills: { snapshot(...): SkillSnapshot };               // prompt + entries + reader
  tools: { resolve(...): AnyAgentTool[]; policy: ToolPolicy };
  model: { stream(preparedTurn) };                        // روی @openclaw/ai یا پیاده‌سازی خودت
  sessions: { append(claim, entries); load(key) };         // writer-claim + transcript
  events: { publish(lifecycle | tool | assistantDelta) };
  hooks: { beforeModelResolve, beforePromptBuild, beforeToolCall };
};
```

مرجع دقیق:

- قرارداد کامل میزبان: `src/agents/harness/host-capability-types.ts`
  (`AgentHarnessHostCapabilities`، حدود ۴۴ عضو)
- نقطهٔ ورود اجرا: `src/agents/embedded-agent-runner/run-orchestrator.ts` (`runEmbeddedAgent`)
  و `src/agents/command/run-embedded-attempt.ts` (`runEmbeddedAgentAttempt`)
- توالی دقیق حلقه: `docs/concepts/agent-loop.md`
- تفکیک runtime / provider / model: `docs/concepts/agent-runtimes.md`

---

## ۵. چه چیزهایی را به مقصد نبر

| حوزه | مسیر |
|---|---|
| Gateway و RPC | `src/gateway` |
| کانال‌ها | `src/channels` |
| خودکارپاسخ/تحویل پیام | `src/auto-reply` |
| رابط کاربری | `ui/` |
| Node host | `src/node-host` |
| سرویس daemon | `src/daemon` |
| زمان‌بندی | `src/cron` |
| Canvas | `src/canvas` |
| Meeting bot | `src/meeting-bot` |
| تولید مدیا | `src/*-generation`، `src/media*` |
| مدیریت پروسه | `src/process` (supervisor/broker) |
| Secrets / Audit | `src/secrets`، `src/audit` |
| حافظه | `src/memory` (اختیاری) |
| لودر پلاگین | `src/plugins` (فقط اگر سیستم پلاگین را نگه می‌داری) |

جایگزین همهٔ این‌ها = همان ۷ درگاه بخش ۴.
اگر لودر پلاگین را نگه می‌داری، `packages/plugin-sdk` مرز عمومی است و طبق `src/plugin-sdk/AGENTS.md`
پلاگین‌ها نباید به `src/agents` یا `src/channels` دسترسی مستقیم داشته باشند.

---

## ۶. ریسک‌های واقعی انتقال

1. **پایداری پیشوند پرامپت**
   ترتیب و شکل ابزارها و متن مهارت‌ها باید قطعی بماند، وگرنه کش پرامپت provider می‌شکند.
   تست مرجع: `src/agents/agent-tools.prefix-stability.test.ts`.

2. **اعتبارسنجی مجدد اختیار بعد از `await`**
   `requestApproval` و `gateAgentToolExecution` باید اختیارِ زنده را از مالک بگیرند، نه فقط شناسهٔ تطبیق‌یافته.

3. **claim نوشتن ترنسکریپت**
   اگر SQLite و `activeWriterRunId` را نمی‌بری، باید مکانیزم معادل «نویسندهٔ فعال» داشته باشی؛
   وگرنه ران‌های هم‌زمان دادهٔ کهنه می‌نویسند — `docs/concepts/agent-loop.md`.

4. **مسیرها و state-dir**
   `src/skills` و `host-skills.ts` به state-dir، workspace و sandbox وابسته‌اند؛
   این‌ها اولین چیزی هستند که در مقصد باید env/port شوند.

5. **مرز redaction**
   لاگ‌ها خام نوشته می‌شوند و فقط در خروجی (export/remote) پاک‌سازی می‌شوند؛ همان قرارداد را نگه دار.

6. **هویت کش**
   هر cache / memo / snapshot باید با همهٔ ورودی‌های مؤثر کلید بخورد (agent، حساب، credential، session، workspace، شکل درخواست).

---

## ۷. ترتیب پیشنهادی مهاجرت

1. `packages/agent-core` + `@openclaw/ai` را با یک هارنس ساده و یک provider راه بینداز.
2. `SkillCatalog` و `ToolRegistry` را با دو-سه ابزار بساز (read / write / exec).
3. یک `SessionStore` ساده (فایل یا SQLite) با claim نوشتن اضافه کن.
4. اعضای `AgentHarnessHostCapabilities` را یکی‌یکی پر کن (`bindToolSurface` → `runBeforeToolCall` → `requestApproval`).
5. در آخر: قلاب‌ها، تأییدها، مدیا/attachment، trajectory.

---

## ۸. چک‌لیست نهایی

- [ ] حلقهٔ مدل از `packages/agent-core` بدون import از `src/*` اجرا می‌شود
- [ ] `SkillSnapshot` از یک provider مستقل تولید می‌شود (بدون state-dir و workspace میزبان)
- [ ] فهرست ابزار قطعی، مرتب و محدودشده با `toolsAllow` است
- [ ] `gateAgentToolExecution` و تأییدها قبل از اثر جانبی دوباره اختیار را می‌سنجند
- [ ] هیچ ابزاری از `gateway`/`channels`/`auto-reply` باقی نمانده است
- [ ] نوشتن ترنسکریپت با claim فعال محافظت می‌شود
- [ ] کش پرامپت با تغییرات بی‌ربط نمی‌شکند (تست prefix-stability معادل)
- [ ] مسیرهای مهارت/وورک‌اسپیس قابل تنظیم (env/config) هستند
