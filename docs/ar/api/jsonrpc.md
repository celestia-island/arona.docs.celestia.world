---
title: "مرجع واجهة JSON-RPC API"
description: "واجهة JSON-RPC 2.0 لمستوى إدارة arona في /api/rpc — طرق الدردشة والوقت الحقيقي والمحرك والمصادقة والمفاتيح والمزوّدين والوكلاء والذاكرة والمحادثات والاستخدام والفوترة والفيديو والنظام عبر HTTP و WebSocket."
---

# مرجع واجهة JSON-RPC API

يُعرّض arona سطح JSON-RPC 2.0 في `/api/rpc` لمستوى الإدارة: المصادقة، المفاتيح، المزوّدين، الوكلاء، الذاكرة، المحادثات، الاستخدام، الفوترة، الفيديو، الوقت الحقيقي والدردشة المتدفقة. إنه يكمّل سطح REST المتوافق مع OpenAI (`/v1/*`، انظر [واجهة REST API المتوافقة مع OpenAI](./openai-rest.md))؛ استخدم REST لأحمال عمل الاستدلال المصادَق عليها بالمفاتيح، وJSON-RPC لإدارة الجلسات/الحسابات والتحكم في البث. يشرح [البدء السريع](../guides/quickstart.md) أول جولة شاملة من البداية إلى النهاية.

يوزّع السطح **39 طريقة طلب** إضافة إلى طريقة بقاء (liveness) واحدة مجهولة خاصة بـ WebSocket فقط، وهي `system.probe` (40 طريقة إجمالًا). كل طلب هو كائن JSON-RPC 2.0 يحمل `jsonrpc: "2.0"` وسلسلة `method` وكائن `params` اختياريًا و`id` اختياريًا.

## النقل

- **HTTP POST `/api/rpc`** — طلب/استجابة. أرسل `Content-Type:
  application/json`. ينتقل JWT في ترويسة `Authorization: Bearer <jwt>`
  يبلغ الحد الأقصى لجسم الطلب 1 ميجابايت.
- **WebSocket `GET /api/rpc`** — اتصال طويل العمر. لا تستطيع المتصفحات ضبط ترويسات مخصصة عند ترقية WebSocket، لذلك ينتقل JWT كمعامل استعلام `?token=<jwt>`؛ يدمجه الخادم في ترويسة `Authorization: Bearer` داخليًا (انظر `packages/core/src/gateway/server.rs`). يمكن للمقابس المصادَق عليها أن تبقى متصلة إلى أجل غير مسمى.
- **الطلبات المجمّعة (batch)** — يُنفَّذ جسم POST الذي يكون مصفوفة JSON عنصرًا عنصرًا ويُجاب عنه بمصفوفة JSON من الاستجابات بنفس الترتيب.
- **الوصول المجهول** — عبر WebSocket دون JWT، تبقى الطرق العامة (`auth.register`/`auth.login`/`auth.refresh`، `providers.list`، `system.status`) قابلة للاستدعاء، ويُجاب عن `system.probe` بإقرار واحد (ack) قبل إغلاق المقبس. تتطلب كل طريقة أخرى JWT صالحًا؛ وتتطلب الطرق المقيدة بالإدارة إضافةً إلى ذلك حساب إدارة (انظر الدليل أدناه). كما تخضع المقابس المجهولة لمهلة خمول 10 ثوانٍ.
- **إرفاق الجلسة** — ترويسة `x-session-id` في `POST /api/rpc` تدفع أيضًا استجابة RPC نفسها إلى قناة تلك الجلسة، إلى جانب إشعارات البث.

## المعرفات (Ids)

تُعاد قيم `id` للطلبات بأمانة النوع: `null` → `null`، والسلاسل → سلاسل، والأعداد الصحيحة → أرقام، وأي شيء آخر (أعداد عائمة، كائنات، أعداد صحيحة خارج نطاق i64) → التمثيل النصي JSON. ويُجاب عن `id` المحذوف بـ `null`.

## إشعارات الخادم → العميل (SSE sidecar)

لا تُسلَّم tokens (tokens) وتقدّم النشر وأحداث الوقت الحقيقي عبر مقبس WebSocket. ينشئ كل RPC متدفق معرّف جلسة ويدفع الإشعارات إلى `GET /api/rpc/events?session=<session_id>` كأحداث مرسَلة من الخادم (server-sent events). اشترك في نقطة SSE **قبل أو فورًا بعد** أن يعيد استدعاء RPC معرّف الجلسة — تُسقَط الإشعارات الصادرة بين عودة الاستدعاء واكتمال إنشاء اشتراك SSE (نافذة ما قبل الاشتراك). النمط الموصى به هو فتح دفق SSE أولًا ثم إطلاق RPC.

طرق الإشعار: `chat.stream` (token واحد لكل حدث من `chat.send`)، و`models.progress` (تقدّم تنزيل نموذج الوكيل من `agents.deploy`)، و`realtime.event` (أحداث الخادم لجلسة وقت حقيقي مفتوحة)، و`video.progress` / `video.done` / `video.failed` (مهام الفيديو غير المتزامنة). راجع الكتالوج الكامل في [الأحداث والإشعارات](./events.md).

## رموز الأخطاء

| الرمز | الاسم | المعنى |
| --- | --- | --- |
| `-32700` | Parse error | جسم الطلب ليس JSON صالحًا. |
| `-32600` | Invalid request | كائن الطلب مشوّه، مثل غياب `method`. |
| `-32601` | Method not found | سلسلة `method` غير معروفة؛ الرسالة تعيدها. |
| `-32602` | Invalid params | فشل إلغاء تسلسل `params` للطريقة. |
| `-32603` | Internal error | إخفاق خادم غير متوقع. |
| `-32000` | `APP_ERROR` | خطأ تطبيق عام — مثل عدم العثور على محادثة/provider/وكيل، أو عدم توفر وكيل متصل للنشر. |
| `-32005` | `AUTH_ERROR` | `"Authentication required"` — JWT مفقود أو غير صالح. وتُستخدم أيضًا بواسطة طرق admin token عندما لا يطابق token الحامل `ARONA_ADMIN_TOKEN` (`"Admin access required"`). |
| `-32006` | `QUOTA_ERROR` | تجاوز الحصة الشهرية للفوترة لطريقة RPC مقيدة بـ JWT (`chat.send`). |
| `-32007` | `ADMIN_REQUIRED` | مستدعٍ مصادَق **غير إداري** لطريقة مقيدة بالإدارة (`agents.*`، `engine.invoke`)؛ تتضمن الرسالة تلميحًا خاصًا بالطريقة. |

> طريقتا `agents.*` و`engine.invoke` للإدارة فقط: تتطلبان JWT
> لحساب يحمل `users.is_admin = true`. يُرفض غير الإداري المصادَق
> بـ `-32007` (`ADMIN_REQUIRED`)؛ ويحصل المستدعي غير المصادَق
> على `AUTH_ERROR` القياسي حتى لا يكشف الخادم
> أن الطريقة مميزة.

## دليل المصادقة (legend)

| الدليل | بيانات الاعتماد |
| --- | --- |
| **public** | لا حاجة لأي بيانات اعتماد. |
| **JWT** | `Authorization: Bearer <jwt>` في HTTP، أو `?token=<jwt>` في WebSocket. |
| **admin (JWT + is_admin)** | JWT حامل لحساب يحمل `users.is_admin = true`. |
| **admin token** | حامل `ARONA_ADMIN_TOKEN` (مُهيَّأ عبر متغير بيئة؛ وعند عدم ضبطه تُرفض الطريقة دائمًا — رفض افتراضي). |

جميع بيانات الاعتماد والعناوين المثالية في هذا المستند هي قيم مؤقتة (عناوين IP توثيقية RFC 5737، ومفاتيح `sk-xxx`). راجع [المصادقة والأمان](../guides/auth-security.md) لنموذج المصادقة الكامل الكامن خلف هذا الدليل.

## الدردشة

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `chat.send` | JWT | `model` (string), `messages` (array of `{ role, content, images?, tool_calls? }`), `temperature?` (number), `max_tokens?` (integer), `conversation_id?` (string), `memory?` (bool), `extra?` (object), `tools?` (array of OpenAI-style function definitions), `provider?` (string) | أرسل جولة دردشة متدفقة. تعيد `{ "stream_id", "memory" }` — `memory` هي حالة الاستدعاء (`enabled` / `disabled` / `offline`)؛ تصل tokens كإشعارات `chat.stream` على الـ sidecar SSE. مع `conversation_id`، يُجمَّع السجل المحفوظ المكتمل من جهة الخادم وتُحفَظ الجولة. مقيدة بالفوترة (الحصة الشهرية → `-32006`)؛ يُسجَّل الاستخدام تحت `jwt-<user-uuid>`. |

## الوقت الحقيقي (جلسات صوت/فيديو ثنائية الاتجاه كاملة)

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `realtime.start` | JWT | `model` (string), `config?` (session config object), `conversation_id?` (string) | افتح جلسة ثنائية الاتجاه كاملة ضد backend الذي يخدم `model`. تعيد `{ "session_id", "stream_session" }`: استخدم `session_id` لـ `realtime.event` / `realtime.stop`، واشترك في `stream_session` على الـ sidecar SSE لتلقي إشعارات `realtime.event`. |
| `realtime.event` | JWT | `session_id` (string), `event` (client event — audio append/commit/clear, image frame, response create/cancel, session stop) | أرسل حدث عميل واحدًا إلى جلسة مفتوحة؛ يُمرَّر إلى backend المنبع. تعيد `{ "ok": true }`. |
| `realtime.stop` | JWT | `session_id` (string) | أغلق جلسة وأزلها. تعيد `{ "removed": bool }`. |

## المحرك (قناة إدراك/تحكم عامة)

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `engine.invoke` | admin (JWT + is_admin) | `model` (string), `method` (string), `params?` (object) | استدعاء طلب/استجابة متزامن لطريقة محرك عشوائية على backend الذي يخدم `model` — القناة عالية التردد لاستدعاءات بأسلوب `sensor.ingest` / `control.setpoint` (حلقات 20–30 هرتز). النتيجة هي استجابة backend الخام. |

## المصادقة

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `auth.register` | public | `email`, `password`, `name?` | سجّل حسابًا. مسموح به فقط بينما يكون التسجيل مفتوحًا (`ARONA_REGISTRATION_OPEN`)؛ يصبح أول مستخدم مسجَّل هو المدير. تعيد نفس استجابة token مثل `auth.login` (`access_token`، `refresh_token`، `token_type`، `expires_in`، `user`). |
| `auth.login` | public | `email`, `password` | سجّل الدخول. تعيد `access_token`، `refresh_token`، `token_type`، `expires_in`، `user` (`{ id, email, name, is_admin }`). محدودة المعدل لكل عنوان IP وحساب. |
| `auth.refresh` | public | `refresh_token` | استبدل refresh token بـ access token جديد (وrefresh token جديد). تُرفض رموز التحديث المعاد استخدامها أو المنتهية بـ `AUTH_ERROR`. |
| `auth.me` | JWT | — | ملف المستخدم الحالي: `{ "id", "email", "name" }`. |

## المفاتيح

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `keys.list` | JWT | — | اسرد API keys الخاصة بالمستدعي (id، name، `key_prefix`، project، طوابع الوقت، علامة النشاط). |
| `keys.create` | JWT | `name`, `project?` | أنشئ API key. تعيد `{ id, name, key, key_prefix, project, created_at }` — يُعرض السر الكامل `arona-<uuid>` في `key` **مرة واحدة**؛ خزّنه فورًا. |
| `keys.revoke` | JWT | `key_id` | ألغِ API key. تعيد `{ "ok": true }`. |

## المزوّدون

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `providers.list` | **public** | — | اسرد المزوّدين المعروفين: الإدخالات الرسمية المدمجة إضافة إلى المخصصة، كبيانات تعريفية للعرض (`id`، `name`، `description`، `website_domain`، `is_official`، `is_operator`). عامة عن قصد — لا تحمل القائمة أي بيانات اعتماد؛ فقط التعديلات أدناه مقيدة بـ JWT. |
| `providers.add` | JWT | `id`, `name`, `description?`, `website_domain?` | أضف إدخال مزوّد مخصصًا. تعيد `{ "ok": true }`. |
| `providers.update` | JWT | `provider_id`, `name?`, `description?`, `website_domain?` | حدّث حقول مزوّد مخصص (الحقول المقدَّمة فقط). تعيد `{ "ok": true }`. |
| `providers.remove` | JWT | `provider_id` | أزل مزوّدًا مخصصًا. تعيد `{ "ok": true }`. |
| `providers.test` | JWT | — | اختبر اتصال مزوّد. كعب (stub): تعيد `{ "ok": true, "message": "Provider connection test not yet implemented" }`. |

## الوكلاء

جميع طرق `agents.*` للإدارة فقط (JWT + `is_admin`). تتصل عُقد الوكلاء باتجاه خارجي عبر `GET /ws/agent`؛ وتتحكم مجموعة RPC هذه في السجل (انظر [عنقود الوكلاء](../guides/agent-cluster.md)).

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `agents.list` | admin (JWT + is_admin) | — | اسرد عُقد الوكلاء المسجَّلة: id، name، host، حالة `online`/`offline` (مبنية على نبضات القلب)، ملخص GPU، النماذج المنشورة، الإصدار، طوابع الوقت. |
| `agents.register` | admin (JWT + is_admin) | `machine_name`, `version` | سجّل عقدة وكيل لدى مدير النفق (tunnel). تعيد `{ "agent_id", "token" }` (token هو بيانات اعتماد مستوى تحكم الوكيل). |
| `agents.deregister` | admin (JWT + is_admin) | `agent_id` | ألغِ تسجيل وكيل (افصلها). تعيد `{ "ok": true }`. |
| `agents.status` | admin (JWT + is_admin) | `agent_id` | حالة كل وكيل: علامة الاتصال، host، ملخص GPU، النماذج المحمّلة، استخدام GPU، طوابع نبضات القلب/الاتصال. |
| `agents.deploy` | admin (JWT + is_admin) | `model_id`, `agent_id?` (empty/missing = least-loaded node; errors if none online) | انشر نموذجًا على وكيل. تعيد `{ "ok": true, "stream_id" }` — اشترك في `stream_id` على الـ sidecar SSE لإشعارات تنزيل `models.progress`. |
| `agents.stop` | admin (JWT + is_admin) | `agent_id`, `model_id` | أوقف نموذجًا منشورًا. تعيد `{ "ok": true, "stream_id": null }` (لا يوجد دفق تقدّم). |

## الذاكرة

تُقدَّم الذاكرة طويلة المدى بواسطة خدمة entelecheia Philia عبر WebSocket؛ لا تحجب الإخفاقات الدردشة أبدًا (انظر [بوابة الذاكرة](../guides/memory-gateway.md)).

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `memory.status` | JWT | — | حالة بوابة الذاكرة: `{ "enabled", "writeback", "events" }` — علامات إضافة إلى ما يصل إلى 50 حدث نشاط حديثًا (الأحدث أولًا). |
| `memory.delete` | JWT | `node_id` | احذف عقدة ذاكرة مخزَّنة. تعيد `{ "deleted": bool }`. |

## المحادثات

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `conversations.list` | JWT | — | اسرد محادثات المستدعي مع طوابع عمر نسبي. |
| `conversations.create` | JWT | `title?` (default `New Conversation`) | أنشئ محادثة. تعيد كائن المحادثة الجديد. |
| `conversations.get` | JWT | `conversation_id` (legacy alias: `id`) | اجلب محادثة مع رسائلها. مع فحص الملكية؛ يُرفض الوصول عبر المستخدمين. |
| `conversations.delete` | JWT | `conversation_id` (legacy alias: `id`) | احذف محادثة (المالك فقط). تعيد `{ "ok": true }`. |

> يقبل `conversations.get` / `conversations.delete` أيضًا مفتاح `id` القديم
> من عملاء لوحة التحكم الأقدم؛ ويفوز `conversation_id`
> عند وجودهما معًا.

## الاستخدام

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `usage.list` | JWT | `limit?` (integer, default 50, clamped to 1–200), `offset?` (integer, default 0), `project?` (string) | سجلات استخدام مقسمة إلى صفحات للمستدعي، الأحدث أولًا، تغطي صفوف API key (بادئة `arona-XX`) وصفوف JWT (`jwt-<user-uuid>`). تعيد `{ "records", "total", "limit", "offset", "project" }`؛ يضيّق مرشح `project` النطاق إلى الصفوف الموسومة بمفاتيح فقط. |

## الفوترة

تُوصف الطبقات والحصص ومحاسبة الاستخدام في [الفوترة والاستخدام](../guides/billing-usage.md).

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `billing.plan` | JWT | — | حالة الفوترة الحالية: `{ "tiers", "current_tier", "usage", "remaining", "quota_exceeded" }` — الاستخدام الشهري (`cost_usd`، tokens، عدد الطلبات) والحصة المتبقية. |
| `billing.plan.set` | admin token | `user_email`, `tier` | اضبط طبقة فوترة المستخدم. تعيد `{ "ok": true }`. تُرفض بـ `AUTH_ERROR` عندما لا يطابق الحامل `ARONA_ADMIN_TOKEN`. |
| `billing.video.pricing.get` | JWT | — | جدول تسعير الفيديو. تعيد `{ "pricing": [...] }`. |
| `billing.video.pricing.set` | admin token | `model`, `mode?` (default `per_second_resolution`), `base_price?` (number, default 0), `price_per_second?` (number, default 0), `price_per_frame?` (number, default 0), `resolution_coeff?` (object), `currency?` (default `USD`), `enabled?` (bool, default `true`) | أدرج أو حدّث تسعير الفيديو لنموذج. تعيد `{ "ok": true }`. تُرفض بـ `AUTH_ERROR` عندما لا يطابق الحامل `ARONA_ADMIN_TOKEN`. |

## الفيديو

مهام توليد الفيديو غير المتزامنة (انظر [الوقت الحقيقي والفيديو](../guides/realtime-video.md)). يُدفع تقدّم المهمة كإشعارات `video.progress` / `video.done` / `video.failed` على قناة الجلسة.

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `video.create` | JWT | `model`, `prompt`, `negative_prompt?`, `images?` (array of `{ data_base64, mime_type }`), `duration_seconds?` (integer), `width?` (integer), `height?` (integer), `provider?` (string), `extra?` (object) | أرسل مهمة توليد فيديو غير متزامنة. تعيد `{ "job_id", "stream_id" }` — اشترك في `stream_id` لإشعارات التقدّم. |
| `video.get` | JWT | `job_id` (UUID) | استطلع حالة/نتيجة مهمة (status، progress، result، error، cost). |
| `video.list` | JWT | `limit?` (integer, default 20) | اسرد مهام المستدعي. تعيد `{ "jobs": [...] }`. |
| `video.cancel` | JWT | `job_id` (UUID) | ألغِ مهمة قيد التشغيل. تعيد `{ "ok": true }`. |

## النظام

| الطريقة | المصادقة | المعاملات | الوصف |
| --- | --- | --- | --- |
| `system.status` | public | — | حالة gateway الكلية: `{ "agents_online", "gpu_nodes", "models_deployed", "requests_total", "requests_per_minute", "uptime_seconds" }`. |
| `system.probe` | anonymous (WS only) | — | فحص بقاء لمرة واحدة عبر نقل WebSocket. يقرّ الخادم بـ `{ "ok": true, "status": "ok" }` ثم يغلق المقبس — لا يحتفظ الزوار المجهولون باتصال مفتوح أبدًا. تُرفض أي طريقة أخرى على مقبس غير مصادَق بـ `AUTH_ERROR`. |

<!-- src: packages/core/src/gateway/rpc.rs:220-397 -->
<!-- note: realtime.start returns {session_id, stream_session} (rpc.rs:1975-1981) — the starter's "→ session_id" was incomplete; stream_session is the SSE channel for realtime.event notifications. -->
<!-- note: realtime.stop returns {removed: bool} (rpc.rs:2032-2033). -->
<!-- note: memory.status returns {enabled, writeback, events} (MemoryStatus, packages/core/src/memory/mod.rs:152-156). -->
<!-- note: agents.register returns {agent_id, token} (tunnel.rs:46-49); agents.stop replies {ok, stream_id: null} (rpc.rs:841-855). -->
<!-- note: usage.list limit is clamped to 1..=200 (rpc.rs:1091); video.list limit default 20 (rpc.rs:1409). -->
<!-- note: auth.register returns the same AuthResponse (tokens + user) as auth.login; the first registered user becomes admin (auth.rs:357). -->
<!-- note: keys.create returns the full arona-<uuid> secret once, in ApiKeyResponse.key (auth.rs:553, 117-125). -->
<!-- note: admin-token methods (billing.plan.set, billing.video.pricing.set) fail with -32005 "Admin access required" when ARONA_ADMIN_TOKEN is unset or the bearer mismatches (rpc.rs:1241-1243, 1444-1446); the env var is optional, default-deny. -->
<!-- note: conversations.get/delete accept a legacy `id` alias besides `conversation_id` (conversation_id_param, rpc.rs:930-941). -->
<!-- note: system.probe acks {ok, status} then closes; anonymous sockets also have a 10 s idle timeout (rpc.rs:149, 156-167, 186-190). -->
