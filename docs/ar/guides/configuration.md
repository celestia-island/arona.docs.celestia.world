---
title: "الإعدادات"
description: "مرجع لكل متغير بيئة يقرؤه خادم أرونا، مع القيم الافتراضية والدلالات."
---

# الإعدادات

يُضبط أرونا **بالكامل عبر متغيرات البيئة**، تُقرأ مرة واحدة عند إقلاع العملية
(`Config::load` في `packages/core/src/config.rs`، بالإضافة إلى عدد قليل يُقرأ
عند أول استخدام). لا يوجد ملف إعدادات: غيّر متغيرًا وأعد تشغيل الخادم
ليُطبَّق.

هذه الصفحة هي المرجع لكل ما يقرؤه كود الخادم، مُجمَّعًا حسب الاهتمام. متغيرات
خاصة بالـ mock ومتغيرات وقت التشغيل مُدرجة من أجل الاكتمال.

## جدول المرجع

| المتغير | القيمة الافتراضية | الغرض |
| --- | --- | --- |
| `DATABASE_URL` | *(مطلوب)* | عنوان اتصال PostgreSQL. |
| `JWT_SECRET` | *(مطلوب خارج وضع الـ mock)* | السر المستخدم لتوقيع الـ JWTs. |
| `ARONA_HOST` | `0.0.0.0` | عنوان الربط (يتراجع إلى `SHITTIM_CHEST_HOST`). |
| `ARONA_PORT` | `8420` | منفذ الربط (يتراجع إلى `SHITTIM_CHEST_PORT`). |
| `ARONA_DATA_DIR` | غير مضبوط | دليل البيانات المحلي. |
| `ARONA_ADMIN_TOKEN` | غير مضبوط | token الحامل (bearer) لمسارات `/api/admin/*` وطرق RPC الإدارية. |
| `ARONA_REGISTRATION_OPEN` | `0` | القيم الصادقة (`1`/`true`/`yes`/`on`) تفتح التسجيل. |
| `ARONA_API_RATE_LIMIT_RPM` | `60` | حد الطلبات في الذاكرة لكل مفتاح في الدقيقة؛ `0` يمنع كل شيء. |
| `MOCK_MODE` | غير مضبوط | الحضور (أي قيمة) يفعّل وضع الـ mock للتطوير. |
| `MOCK_SEED_PATH` | غير مضبوط | ملف بذر SQL خام يُنفَّذ في وضع الـ mock. |
| `ARONA_MEMORY_URL` | غير مضبوط | عنوان WebSocket لبوابة ذاكرة Philia. |
| `ARONA_MEMORY_TOKEN` | غير مضبوط | token لبوابة الذاكرة. |
| `ARONA_MEMORY_WRITEBACK` | `true` | كتابة جولات الدردشة المكتملة إلى الذاكرة؛ يقبل `true`/`false` (أي قيمة أخرى تتراجع إلى الافتراضي). |
| `ARONA_AGENT_NAME` | `arona-agent` | هوية وكيل عُقدة GPU. |
| `ARONA_PANEL_URL` | `localhost:8080` | حيث يتصل الوكيل (`ws://<panel_url>/ws/agent`). |
| `ARONA_EVERNIGHT_URL` | `ws://127.0.0.1:3001/ws` | وكيل evernight المحلي لعناوين backend من نوع `evernight://`. |
| `ARONA_MISTRALRS` | غير مضبوط | الحضور يفرض محرك Mistral.rs لخطط نماذج Gguf. |
| `ARONA_AGENT_BIND_ADDR` | `127.0.0.1` | الواجهة التي يربط بها خادم نموذج llama.cpp المُنشأ. |
| `HF_ENDPOINT` | `https://huggingface.co` | عنوان Hugging Face الأساسي لتنزيلات النماذج. |
| `GITHUB_TOKEN` | غير مضبوط | token الوصول لسجل نماذج GitHub. |
| `RUST_LOG` | غير مضبوط | عامل تصفية التتبّع، مثل `info` أو `arona=debug,info`. |

## المتغيرات المطلوبة

### `DATABASE_URL`

عنوان اتصال PostgreSQL. **مطلوب**: يخرج الخادم برسالة
`FATAL: DATABASE_URL not set. arona requires a PostgreSQL database.` عندما
يكون فارغًا، ويرفض الأمر الفرعي `migrate` في الـ CLI العمل. يُنشأ المخطط
/ يُحدَّث تلقائيًا عبر `serve` عند الإقلاع.

### `JWT_SECRET`

السر المستخدم لتوقيع أزواج JWT للوصول/التحديث التي يصدرها `auth.login`
و`auth.register`. **مطلوب في الإنتاج**: يضمّن الكود سر تطوير احتياطيًا
(`dev-secret-not-for-production-use-only-32chars`)، لكن الخادم يرفض الإقلاع
به إلا إذا كان `MOCK_MODE=1`:

```
FATAL: JWT_SECRET environment variable must be set.
Refusing to serve with the built-in development secret;
set MOCK_MODE=1 only for local development.
```

استخدم قيمة طويلة وعشوائية (مثل `openssl rand -hex 32`).

## الخادم

### `ARONA_HOST` / `ARONA_PORT`

عنوان الربط والمنفذ للـ gateway. للتوافق مع الإصدارات القديمة يتراجعان إلى
`SHITTIM_CHEST_HOST` / `SHITTIM_CHEST_PORT`؛ الافتراضيات النهائية هي
`0.0.0.0:8420`.

### `ARONA_DATA_DIR`

دليل بيانات محلي اختياري، يُحمَل على حالة التطبيق للمكوّنات التي تحتاج
موقعًا مؤقتًا للعمل. غير مضبوط افتراضيًا.

## الأمان والتحكم في الوصول

### `ARONA_ADMIN_TOKEN`

token الحامل (bearer) الذي يحرس مسارات HTTP `/api/admin/*`
(`POST/GET/DELETE /api/admin/backends`، `/api/admin/aliases`) وطرق RPC
`billing.plan.set` / `billing.video.pricing.set`. عندما يكون **غير مضبوط**،
يرفض كل واحد من تلك المسارات برسالة `Admin access required` (401) — لا يوجد
افتراضي. اضبطه على قيمة عشوائية قوية قبل بدء الخادم.

### `ARONA_REGISTRATION_OPEN`

يفتح التسجيل الذاتي عبر `auth.register`. القيم الصادقة هي بالضبط `1`، `true`،
`yes`، `on` (غير حساسة لحالة الأحرف)؛ أي شيء آخر — بما فيه `0`، `false`،
`off`، أو متغير غير مضبوط/فارغ — يبقى مغلقًا. يُقرأ العلم مرة واحدة عند
الإقلاع. **أول مستخدم مُسجَّل مسموح له دائمًا** (حتى عندما يكون التسجيل مغلقًا)
ويصبح المدير.

### `ARONA_API_RATE_LIMIT_RPM`

حد معدل نافذة منزلقة في الذاكرة لكل مفتاح (طلبات في الدقيقة)، يُطبَّق على كل
طلب `/v1/*` مصادَق عليه (الدردشة، embeddings، الفيديو، النماذج)، مُفهرَسًا
بـ hash مفتاح API (أو تسمية `u:<email>` لمسار `GET /v1/models` الذي يقبل
JWT). لا يخضع سطح RPC لهذا المحدد — مستخرجات مصادقة `/v1/*` هي المستدعيات
الوحيدة. الافتراضي `60`. تُحلَّل القيمة مرة واحدة في `OnceLock` على مستوى
العملية. **قيمة `0` تمنع كل طلب** — الفحص هو `entry.len() >= rpm`، لذا مع
`0` لا يمكن لأي طلب المرور. هذا هو الحد على مستوى الـ gateway؛ تفرض مستويات
الفوترة حدودها الخاصة لكل مفتاح فوقه.

## التطوير

### `MOCK_MODE`

وضع الـ mock للتطوير، يُفعَّل **بالحضور** — الفحص هو
`std::env::var("MOCK_MODE").is_ok()`، لذا فإن *أي* قيمة (بما فيها `0` أو
قيمة مضبوطة وفارغة) تفعّله. وهو:

- يرفع شرط `JWT_SECRET` (يصبح سر التطوير المدمج مقبولًا)؛
- يبذر أربعة حسابات تجريبية (`demiurge@celestia.world`، `momoi@celestia.world`،
  `midori@celestia.world`، `yuzu@celestia.world`، كلمة المرور `33550336`)؛
- ينتظر اكتمال البذر قبل ربط المستمع.

لا تستخدمه أبدًا في الإنتاج.

### `MOCK_SEED_PATH`

في وضع الـ mock فقط، يشير إلى ملف SQL خام يُنفَّذ بدلًا من بذر الحسابات
المدمج (تُقسَّم العبارات على `;`، وتُتخطّى تعليقات `--`). إذا تعذّرت قراءة
الملف، يُستخدم البذر المدمج كاحتياط.

## بوابة الذاكرة

### `ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` / `ARONA_MEMORY_WRITEBACK`

إعدادات بوابة الذاكرة طويلة المدى (entelecheia Philia). الذاكرة **معطّلة
تمامًا** إلا إذا كان كل من `ARONA_MEMORY_URL` و`ARONA_MEMORY_TOKEN` مضبوطين
وغير فارغين. عند التفعيل:

- تُسترجَع جولات الدردشة المكتملة وتُحقن كسياق، و
- يتحكم `ARONA_MEMORY_WRITEBACK` (الافتراضي `true`) فيما إذا كانت الجولات
  المكتملة تُكتب مرة أخرى إلى خدمة الذاكرة؛ `0` أو `false` يعطّل الكتابة
  العكسية.

لا تحجب أعطال الذاكرة الدردشة أبدًا؛ تنعكس الحالة الناتجة في ترويسة الاستجابة
`X-Arona-Memory` (`enabled` / `disabled` / `offline`).

## هوية الوكيل ومجموعة الوكلاء

### `ARONA_AGENT_NAME` / `ARONA_PANEL_URL`

هوية ملف الوكيل الثنائي لعُقدة GPU (`_agent`): يُبلَّغ `ARONA_AGENT_NAME`
(الافتراضي `arona-agent`) إلى اللوحة كاسم/معرّف الوكيل، و`ARONA_PANEL_URL`
(الافتراضي `localhost:8080`) هو المكان الذي يتصل منه الوكيل
(`ws://<panel_url>/ws/agent`).

واجهة HTTP الخاصة بالوكيل **مشفّرة بشدة** لترتبط بـ `0.0.0.0:5790` — لا يوجد
متغير بيئة لعنوان الربط الخاص بها.

### `ARONA_AGENT_BIND_ADDR`

الواجهة التي يربط بها **خادم نموذج llama.cpp المُنشأ** عندما ينشر الوكيل
نموذج Gguf، حتى يمكن الوصول إلى المحرك من أجهزة أخرى (مثل `0.0.0.0`).
الافتراضي `127.0.0.1`. لاحظ أن هذا *ليس* ربط واجهة HTTP للوكيل (المثبّت عند
`0.0.0.0:5790`).

## جسر Evernight

### `ARONA_EVERNIGHT_URL`

عنوان WebSocket لوكيل evernight المحلي المستخدم لحلّ عناوين backend من نوع
`evernight://` إلى إعادة توجيه TCP محلية. الافتراضي `ws://127.0.0.1:3001/ws`.

## بيئة تشغيل النماذج والتنزيلات

### `ARONA_MISTRALRS`

الحضور (أي قيمة) يفرض محرك Mistral.rs لخطط نماذج Gguf التي كانت ستفترض
افتراضيًا llama.cpp. دلالات الحضور مثل `MOCK_MODE`.

### `HF_ENDPOINT`

العنوان الأساسي لتنزيلات نماذج Hugging Face (مصادر `hf:`)، الافتراضي
`https://huggingface.co`. اضبطه على مرآة مثل `https://hf-mirror.com` عندما
يكون huggingface.co غير قابل للوصول. يقرؤه مُنزِّل النماذج؛ تُقتطع الشرطة
المائلة الزائدة.

### `GITHUB_TOKEN`

token الوصول الذي يستخدمه سجل نماذج GitHub (مصادر `gh:`) للوصول إلى API.
غير مضبوط افتراضيًا؛ بدونه تنطبق حدود معدل GitHub API.

## التسجيل

### `RUST_LOG`

عامل تصفية تتبّع قياسي يطبّقه `tracing_subscriber` عند الإقلاع، مثل
`info` أو `arona=debug,info`. يتبع دلالات `RUST_LOG` المعتادة
(`error`/`warn`/`info`/`debug`/`trace`، وتجاوزات لكل هدف).

## الافتراضات في لمحة

| الإعداد | الافتراضي |
| --- | --- |
| عنوان / منفذ الربط | `0.0.0.0:8420` |
| حد معدل API لكل مفتاح | 60 RPM |
| اسم الوكيل | `arona-agent` |
| عنوان اللوحة | `localhost:8080` |
| كتابة الذاكرة العكسية | مفعّلة |
| التسجيل | مغلق |

<!-- src: packages/core/src/config.rs (Config::load), packages/core/src/gateway/run.rs:40-50 (startup checks), packages/core/src/auth.rs:30-38,160-214,694-705 (rate limit, mock seed), packages/core/src/gateway/server.rs:76-79 (data dir, admin token), packages/core/src/memory/mod.rs:79-98, packages/core/src/backends/bridge.rs:74-82, packages/core/src/pipeline.rs:136, packages/core/src/orchestration/llama_cpp.rs:434-451, packages/core/src/models/download.rs:30-40, packages/agent/src/main.rs:109 -->
<!-- note: beyond the fact sheet, four more variables are read by the server code and were added to the table for completeness: ARONA_EVERNIGHT_URL (backends/bridge.rs:76, default ws://127.0.0.1:3001/ws), ARONA_MISTRALRS (pipeline.rs:136, presence semantics), ARONA_AGENT_BIND_ADDR (orchestration/llama_cpp.rs:437-438, default 127.0.0.1 — the bind of the spawned llama.cpp engine, distinct from the agent HTTP API which stays hardcoded to 0.0.0.0:5790, agent/src/main.rs:109), and HF_ENDPOINT (models/download.rs:30-40). -->
