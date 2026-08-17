---
title: "مجموعة الـ agents"
description: "مجموعات GPU متعددة العقد — تنزيل أوزان النماذج عبر الـ CLI، وتشغيل ملف _agent التنفيذي على عقد GPU، وقيادة عمليات النشر عبر سطح RPC الخاص بـ agents.*."
---

# مجموعة الـ agents

تنقسم قصة نشر arona إلى نصفين. **اللوحة (panel)** (ملف `arona` التنفيذي للخادم) تملك التوجيه والفوترة والمصادقة ومستوى الإدارة. تشغّل كل عقدة GPU عملية **`_agent`** واحدة تملك أوزان النماذج وعمليات الخدمة المحلية. تفتح الـ agents اتصال WebSocket طويل العمر عائدًا إلى مستوى التحكم بالـ agents في اللوحة (`/ws/agent`)؛ وتدفع اللوحة أوامر `deploy` / `stop` عبر ذلك المقبس، بينما يبث الـ agent تقدم التنزيل ونبضات القلب ونتائج الأوامر عائدًا إلى الأعلى. وبمجرد تشغيل نموذج على أحد الـ agents، تسجّله اللوحة كـ backend قابل للتوجيه ليصل إليه كل من حركة `/v1/*` وRPC — مستوى التحكم عبر WebSocket، بينما مستوى البيانات عبر HTTP عادي إلى منفذ المحرك المحلي في الـ agent.

## تنزيل أوزان النماذج (CLI)

ينزّل ملف `_cli` التنفيذي أوزان النماذج من HuggingFace أو ModelScope أو إصدارات GitHub:

```bash
arona download <repo> [--filter <glob|prefix>]... [--out <dir>] [--revision <rev>] [--yes]
```

- **أشكال الـ repo** — `hf:owner/repo` (الافتراضي؛ و`owner/repo` المجرد يُحل أيضًا إلى HuggingFace)، و`ms:owner/repo` (ModelScope)، و`gh:owner/repo[:tag]` (إصدار GitHub، الوسم اختياري). البادئات الطويلة `huggingface:` و`modelscope:` و`github:` مقبولة أيضًا؛ أما المعرّف المجرد بلا شرطة مائلة فيُحل إلى سجل Ollama (`packages/core/src/models/download.rs:21-28,55-86`).
- **`--filter <glob|prefix>`** — قابل للتكرار؛ تُنزَّل فقط ملفات القائمة (manifest) المطابقة للنمط glob (أو البادئة). وبدون فلتر يُحدد **الـ repo كاملًا**.
- **التأكيد** — يطلب التنزيل غير المفلتر دائمًا `Continue? [y/N]` قبل البدء ما لم يُمرَّر `--yes`. أما التنزيل المفلتر فلا يطلب تأكيدًا أبدًا؛ وعندما يبلغ المجموع المحدد 2 GiB أو أكثر يطبع لافتة معلوماتية بدلًا من ذلك (`NO_CONFIRM_THRESHOLD`, `packages/cli/src/main.rs:12-15, 439-464`).
- **`--out <dir>`** — يلغي الوجهة الافتراضية `~/.arona/models/<repo-id>`.
- **`--revision <rev>`** — يلغي أي لاحقة `:rev` مضمّنة (`hf:owner/repo:rev`).
- **الاستئناف** — تُستأنف التنزيلات المتقطعة تلقائيًا: يُحتفظ بملف `.part` ويستمر التنزيل من طوله الحالي عبر طلب Range؛ وتُتخطى الملفات المكتملة حسب الحجم، وعندما تحمل القائمة (manifest) بصمة (digest) تُتحقق منها عبر SHA-256 (`packages/cli/src/main.rs` `verify_sha256`/`summarize`).
- **إعادة المحاولة** — تُعاد محاولة أخطاء الشبكة حتى 3 مرات مع تأخير 5 ثوانٍ؛ بينما تفشل الأخطاء غير الشبكية فورًا (`packages/cli/src/main.rs:277-302`).
- **`HF_ENDPOINT`** — يبدّل عنوان HuggingFace الأساسي، مثلًا إلى مرآة (mirror):

  ```bash
  HF_ENDPOINT=https://hf-mirror.com arona download hf:owner/repo --filter "*.safetensors"
  ```

أوامر الـ CLI الأخرى (`packages/cli/src/main.rs:28-53`):

| الأمر | الغرض |
| --- | --- |
| `install` | إعداد بيئة بنقرة واحدة: يكتشف ملف العتاد (hardware profile) ويطبع توصيات الـ backend / التكميم (quantization). |
| `status` | يطبع ملف العتاد (hardware profile). |
| `deploy <model>` | يحل نموذجًا محليًا ويُبلغ عما إذا كان مخزَّنًا مؤقتًا (cached) بالفعل. |
| `download` | تنزيل أوزان النماذج (أعلاه). |
| `serve` | يبدأ خادم الـ API (اللوحة). |
| `connect <url>` | يتصل بلوحة إدارة. |
| `migrate` | يشغّل ترحيلات قاعدة البيانات. |

## ملف `_agent` التنفيذي

يعمل `_agent` على كل عقدة GPU ويُكوَّن حصريًا عبر متغيرات البيئة (`packages/core/src/config.rs:37-40`):

| المتغير | الافتراضي | المعنى |
| --- | --- | --- |
| `ARONA_AGENT_NAME` | `arona-agent` | معرّف عقدة فريد؛ تستخدمه اللوحة كـ `agent_id`. |
| `ARONA_PANEL_URL` | `localhost:8080` | `host:port` الخاص باللوحة؛ يتصل الـ agent بـ `ws://{ARONA_PANEL_URL}/ws/agent`. |

انظر [الإعدادات](./configuration.md) للمرجع الكامل لمتغيرات البيئة (متغيرات جانب اللوحة، قاعدة البيانات، الأسرار).

```bash
# On each GPU node:
export ARONA_AGENT_NAME="gpu-node-01"
export ARONA_PANEL_URL="192.0.2.10:8420"   # the panel's host:port
_agent
```

السلوك:

- **اتصال التحكم** — يعيد الـ agent الاتصال بـ `ws://{ARONA_PANEL_URL}/ws/agent` (`packages/agent/src/panel.rs:23`). وعند الاتصال يرسل إطار (frame) `register` يحمل `agent_name` و`gpu_info` وقائمة النماذج المنشورة مسبقًا؛ وتسجّل اللوحة عنوان نظير TCP للـ agent كقيمة `host` له.
- **التراجع عند إعادة الاتصال (backoff)** — يبدأ من ثانية واحدة ويتضاعف حتى سقف 60 ثانية (`packages/agent/src/panel.rs:27,33-34,63`).
- **نبضات القلب** — كل 30 ثانية يُبلغ الـ agent عن استخدام GPU وعدد النماذج المحمَّلة ومدة التشغيل (uptime). وتعتبر اللوحة الـ agent غير متصل عندما يتجاوز عمر آخر نبضة قلب 30 ثانية.
- **واجهة HTTP المحلية** — يربط العنوان **الثابت** `0.0.0.0:5790`؛ ولا يوجد متغير بيئة لعنوان الربط (`packages/agent/src/main.rs:109`). تدمج اللوحة هذا المنفذ مع الـ host المسجَّل للـ agent لبناء عنوان مستوى البيانات للنماذج المنشورة.
- **الأوامر** — تضع اللوحة أوامر `deploy` / `stop` في قائمة انتظار عبر المقبس. يحمل أمر `deploy` قيمة `model_id` و`stream_id`؛ ويُبث تقدم التنزيل عائدًا كإطارات `deploy_progress` على نفس المقبس (تحوّلها اللوحة إلى إشعارات `models.progress` عبر SSE، انظر [الأحداث والإشعارات](../api/events.md))، ويُبلِّغ إطار `deploy_result` النهائي عن منفذ المحرك المحلي `port` و`backend`. ويُجاب عن `stop` بإطار `stop_result`.

شغّل `_agent` تحت مشرف خدمات (systemd، malkuth، ...) ليعيد الاتصال تلقائيًا؛ وتتحمل اللوحة إعادة التشغيل من أي من الجانبين (انظر [استمرارية العقد](#node-persistence) أدناه).

## RPC لمستوى التحكم بالـ agents

سطح الـ agents بالكامل محكوم بالإدارة (admin-gated): يتطلب كل أسلوب JWT صالحًا **و** حساب مدير (`validate_admin_jwt` يفحص `is_admin_email`؛ `packages/core/src/gateway/rpc.rs:106-118,301-337`).

| الأسلوب | المعاملات | النتيجة |
| --- | --- | --- |
| `agents.list` | — | طوبولوجيا المجموعة (cluster): `id`، `name`، `host`، `status` (`online`/`offline`)، ملخص GPU، `models`، `last_heartbeat`، `version`، `connected_at`. |
| `agents.register` | `machine_name`, `version` | `agent_id`، `token`. |
| `agents.deregister` | `agent_id` | `{ "ok": true }` — يزيل العقدة. |
| `agents.status` | `agent_id` | `online`، ملخص GPU، `gpu_utilization`، `models`، `host`، `connected_at`، `last_heartbeat`. |
| `agents.deploy` | `model_id`, `agent_id?` | `{ "ok": true, "stream_id" }` — قيمة `agent_id` فارغة تستهدف تلقائيًا العقدة الأقل تحميلًا. |
| `agents.stop` | `agent_id`, `model_id` | `{ "ok": true }` — يوقف النشر. |

يُرجع `agents.deploy` قيمة `stream_id`؛ اشترك في `/api/rpc/events?session=<stream_id>` **قبل** الاستدعاء أو فورًا بعده لتلقي إشعارات تنزيل `models.progress` (انظر [الأحداث والإشعارات](../api/events.md)). وانظر [واجهة JSON-RPC API](../api/jsonrpc.md) لتفاصيل النقل والمصادقة.

## التسجيل التلقائي للنماذج المنشورة

عندما يُبلِّغ إطار `deploy_result` عن النجاح، تسجّل اللوحة `ExternalApiBackend` باسم **`agent-{model_id}`** في موجّه الـ gateway، بعنوان أساسي `http://{agent-host}:{port}` — أي الـ host المسجَّل للـ agent بالإضافة إلى منفذ المحرك الذي أبلغ عنه (`packages/core/src/gateway/server.rs:310-366`, `packages/core/src/gateway/mod.rs:253-270`). يصبح النموذج المنشور backend عاديًا قابلًا للتوجيه: تصل إليه `/v1/chat/completions` والتضمينات (embeddings) ودردشة RPC، وتُطبَّق الأسماء المستعارة (aliases)، ويستقصيه الـ health checker (انظر [الـ backends](./backends.md) لأنواع الـ backends ودلالات الفحص).

- إعادة نشر النموذج نفسه (مثلًا على agent مختلف) يستبدل الـ backend السابق.
- إطار `stop_result` ناجح يلغي تسجيله مرة أخرى (`packages/core/src/gateway/mod.rs:274-287`)؛ ويتوقف معرّف النموذج عن الحل (resolving).

## التنسيب

تمر عمليات النشر بلا `agent_id` صريح عبر تنسيب الأقل تحميلًا (least-loaded) (`packages/core/src/gateway/tunnel.rs:214-229`): المرشحون هم الـ agents الذين يقل عمر آخر نبضة قلب لديهم عن 30 ثانية، ويُختار ذو **أدنى متوسط استخدام لـ GPU**. أما الـ agents بلا بيانات قياس (telemetry) فيُرتَّبون أخيرًا لكنهم يبقون قابلين للاختيار. وإذا لم يكن أي agent متصلًا، يفشل RPC برسالة `No online agents available for deployment`.

في جانب التوجيه، تُثبَّت المحادثات **على backend واحد** (تقارب الجلسة): يُسجَّل أول backend يخدم محادثة ويُعاد استخدامه في الأدوار اللاحقة، فتبقى حالة كل محادثة مثل ذاكرة التخزين المؤقت KV قيد التشغيل دافئة (`packages/core/src/routing/mod.rs:31-34,110-138`). وإذا أصبح الـ backend المثبَّت غير سليم، يتراجع التوجيه إلى اختيار جديد ويعيد التثبيت.

## استمرارية العقد

تُحفظ عقد الـ agents في جدول `agent_nodes` (`agent_id`, `machine_name`, `version`, `host`, `gpu_info`, `models`, `connected_at`, `last_heartbeat`؛ `packages/core/src/gateway/tunnel.rs:8-12`). عند تشغيل اللوحة تُستعاد الصفوف المحفوظة فتبقى العقد المسجَّلة سابقًا ظاهرة عبر عمليات إعادة التشغيل؛ وتكون الإدخالات المستعادة **بلا مرسل (sender-less)** حتى يعيد كل agent الاتصال عبر WebSocket (`packages/core/src/gateway/run.rs:74-85`). لذلك يفشل النشر إلى عقدة مستعادة لكنها غير متصلة حتى يعيد `_agent` الخاص بها الاتصال.

<!-- note: the fact sheet said `arona install` "installs the server binary" — the current code performs hardware detection and prints backend/quantization setup recommendations (packages/cli/src/main.rs:83-113). -->
<!-- note: the fact sheet said unfiltered downloads "must be confirmed when over the 2 GiB threshold" — unfiltered downloads are always confirmed unless --yes is passed; the 2 GiB NO_CONFIRM_THRESHOLD only gates an informational banner when --filter is given (packages/cli/src/main.rs:439-464). -->
<!-- src: packages/core/src/gateway/tunnel.rs:8-229 (agent registry, persistence, placement) -->
