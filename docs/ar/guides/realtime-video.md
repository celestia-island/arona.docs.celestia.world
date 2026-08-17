---
title: "الوقت الفعلي والفيديو"
description: "جلسات الوقت الفعلي ثنائية الاتجاه الكامل (realtime.start/event/stop)، وقناة الإدراك والتحكم engine.invoke، ومهام توليد الفيديو غير المتزامنة."
---

# الوقت الفعلي والفيديو

تكشف أرونا عن قناتين متعددتي الوسائط إلى جانب الدردشة النصية العادية: **جلسات
الوقت الفعلي ثنائية الاتجاه الكامل** (إدخال وإخراج الكلام والفيديو عبر قناة
ثنائية الاتجاه واحدة) و**توليد الفيديو غير المتزامن** (مهام بنمط المهام تعمل
في الخلفية وتبلغ عن التقدم). تُوجَّه كلتاهما إلى الـ backend الذي يخدم النموذج
المطلوب وتُحتسب كلتاهما عبر طبقة الفوترة.

## جلسات الوقت الفعلي

جلسة الوقت الفعلي هي قناة ثنائية الاتجاه بين **عميل واحد** و**مزوّد علوي واحد**
(upstream): واجهة برمجة وقت فعلي سحابية (مفردات WebSocket الخاصة بـ
OpenAI-Realtime / Qwen-Omni-Realtime) أو محرك CEP محلي. تصل أحداث العميل عبر
JSON-RPC وتُمرَّر إلى الأعلى؛ وتُدفع أحداث الخادم إلى الخلف كإشعارات
`realtime.event` عبر قناة الجلسة SSE. ينتقل الصوت بصيغة base64 PCM16
(16 kHz من العميل إلى النموذج، و24 kHz من النموذج إلى العميل)، مطابقًا لصيغة
النقل عند باعة السحابة بحيث يمرر الـ gateway البايتات كما هي دون تعديل
(`packages/core/src/backends/realtime.rs:1-19`).

### `realtime.start`

يفتح جلسةً ضد الـ backend الذي يخدم `model` (JWT؛ المعاملات `model`،
`config?`، `conversation_id?` — `packages/core/src/gateway/rpc.rs:1890-1898,
1914-1984`). يجب على الـ backend **أن يصرّح** بالقدرة `realtime` (وسائط الصوت
والفيديو)؛ وإلا يفشل الاستدعاء صراحةً برسالة
`model {model} does not support realtime sessions (no audio/video modality)` —
ولا يوجد أي تراجع صامت إلى الدردشة النصية
(`packages/core/src/gateway/realtime.rs:138-142`).

يُدعم نوعان من الـ upstream (`packages/core/src/gateway/realtime.rs:143-167`):

- **محرك CEP upstream** — يوجّه الأحداث عبر قناة البث `Engine.InvokeStart`
  الخاصة ببروتوكول محرك Celestia، بحيث ينضم محرك omni منشور محليًا إلى نفس
  واجهة العميل دون صيغة نقل جديدة.
- **upstream سحابي** — عنوان `wss://` ثابت يتحدث بمفردات أحداث الوقت الفعلي
  السحابية (`session.update`، `input_audio_buffer.*`، `response.audio.delta`،
  ...). يعيد التنفيذ السحابي إرسال `session.update` عند إعادة الاتصال.

الاستجابة هي `{ "session_id": ..., "stream_session": ... }` — اشترك في
`/api/rpc/events?session=<stream_session>` قبل الاستدعاء (أو فورًا بعده) لتلقي
أحداث الخادم. يُثبّت `conversation_id` الاختياري نسخة الكلام المنطوق كرسائل
مساعد ويسجّل استهلاك الـ token لأغراض الفوترة
(`packages/core/src/gateway/realtime.rs:32-85`).

### `realtime.event`

يرسل حدث عميل واحدًا إلى الجلسة (JWT؛ المعاملان `session_id`، `event` —
`packages/core/src/gateway/rpc.rs:1989-2013`). تشمل الأحداث المدعومة
`session.update`، و`input_audio_buffer.append` / `.commit` / `.clear`،
و`input_image_buffer.append`، و`response.create`، و`response.cancel`، و
`session.stop`. إن `send_event` **غير محجوب (non-blocking)**: يُدرج الحدث في
قائمة انتظار على قناة mpsc وتكتبه مهمة الإعادة إلى الـ upstream
(`packages/core/src/gateway/realtime.rs:254-280`). ولا يجوز لأحد سوى مالك
الجلسة إرسال الأحداث.

### `realtime.stop`

يغلق الجلسة ويزيلها (JWT؛ المعامل `session_id` —
`packages/core/src/gateway/rpc.rs:2016-2034`). تملك كل جلسة **مهمة إعادة
واحدة (forwarder task)** تمسك الـ upstream وتعدد الإرسال في الاتجاهين معًا:
تُستهلك أحداث العميل من قائمة الانتظار وتُستطلَع أحداث الـ upstream في
الحلقة نفسها. تخرج مهمة الإعادة عند إغلاق الـ upstream أو عند إيقاف الجلسة،
مزيلةً إدخال السجل
(`packages/core/src/gateway/realtime.rs:195-250`).

تُدفع أحداث الخادم كإشعارات `realtime.event` مع المعاملين
`{ session_id, event }` عبر قناة الجلسة — انظر
[الأحداث والإشعارات](../api/events.md).

## `engine.invoke`

إن `engine.invoke` هو قناة أساليب المحرك العامة **المتزامنة**
(ADMIN: JWT + `is_admin`؛ المعاملات `model`، `method`، `params?` —
`packages/core/src/gateway/rpc.rs:261-264,2049-2079`). يستدعي أسلوبًا عشوائيًا
على الـ backend الذي يخدم `model` ويعيد النتيجة مباشرة، ما يجعله قناة
الإدراك/التحكم عالية التردد: استدعاءات بنمط `sensor.ingest` و`control.setpoint`
في حلقات بتردد 20-30 Hz. ترفض الـ backends التي لا تملك قناة استدعاء عامة
(كل الـ backends المتوافقة مع HTTP الخاصة بـ OpenAI) صراحةً برسالة
`backend does not support generic invocation`
(`packages/core/src/backends/mod.rs:573-586`).

## توليد الفيديو (REST)

مهام الفيديو هي مهام غير متزامنة بنمط OpenAI عبر سطح REST (مصادقة API key —
`packages/core/src/gateway/server.rs:876-993`؛ انظر
[واجهة REST API المتوافقة مع OpenAI](../api/openai-rest.md)):

**`POST /v1/video/generations`**

| الحقل | النوع | ملاحظات |
| --- | --- | --- |
| `model` | string | مطلوب — يختار backend يدعم الفيديو. |
| `prompt` | string | مطلوب. |
| `negative_prompt` | string? | |
| `images` | array? | عناوين بيانات Base64 URL (`data:image/png;base64,...`)، تُحمَل كـ `{ data_base64, mime_type }`. |
| `duration_seconds` | int? | |
| `width` / `height` | int? | |
| `provider` | string? | تلميح اختيار الـ backend (يُطابَق مع اسم الـ backend). |
| `extra` | object? | تجاوزات خاصة بالـ backend (seed، steps، cfg، ...). |

الاستجابة:

```json
{
  "id": "<job_id>",
  "object": "video.generation",
  "model": "<model>",
  "status": "queued",
  "created_at": 1750000000
}
```

يستطلع **`GET /v1/video/generations/{id}`** المهمة ويعيد `id`، `object`،
`model`، `status`، `progress`، `result`، `error`، `cost`، `created_at`. تُقيَّد
المهام بنطاق المستدعي: أي مهمة يملكها مستخدم آخر تُعيد 404. يفرض سطح REST نفس
بوابات الفوترة (الحصة الشهرية، حد المعدل لكل دقيقة) التي يفرضها مسار الدردشة.

## توليد الفيديو (RPC)

تتوفر القدرة نفسها عبر JSON-RPC (JWT —
`packages/core/src/gateway/rpc.rs:371-386,1296-1431`):

| الأسلوب | المعاملات | النتيجة |
| --- | --- | --- |
| `video.create` | نفس حقول استدعاء REST | `{ job_id, stream_id }`. |
| `video.get` | `job_id` | عرض المهمة (status، progress، result، cost، ...). |
| `video.list` | `limit?` (الافتراضي 20، محصور بين 1-100) | `{ jobs: [...] }`، الأحدث أولًا. |
| `video.cancel` | `job_id` | `{ "ok": true }` — لا يجوز الإلغاء إلا للمالك. |

يعيد `video.create` قيمة `stream_id`؛ اشترك في
`/api/rpc/events?session=<stream_id>` لتلقي إشعارات المهمة
(`video.progress` / `video.done` / `video.failed` — انظر
[الأحداث والإشعارات](../api/events.md)).

## الـ Backend

توليد الفيديو **سحابي فقط**: واجهة برمجة منصة MiniMax H3 المفتوحة، بنوع
backend `minimax-cloud` (`BackendKind::CloudVideo` —
`packages/core/src/backends/mod.rs:502-504,720-727,759-761`). التدفق بنمط
المهام:

1. `POST /v1/video_generation_v2` → `task_id`
2. استطلاع `GET /v1/query/video_generation_v2?task_id=...` حتى `Success` /
   `Fail` / أو بقاء `Pending`
3. عند النجاح، استحضار الناتج (artifact) عبر
   `GET /v1/files/{file_id}/content` → `{ file: { download_url } }`

(`packages/core/src/backends/minimax_cloud.rs:66-210`). لا يخدم backend
MiniMax الدردشة والتضمينات (chat/embeddings)؛ وتصرّح قدراته بـ
`supports_video_generation` و`realtime: false` (انظر
[الـ Backends](./backends.md) لنموذج القدرات). لا يحل التوجيه (routing)
طلبات الفيديو إلا ضد الـ backends التي تملك `supports_video_generation`،
مراعيًا تلميح `provider` الاختياري
(`packages/core/src/routing/mod.rs:205-263`).

**أُزيل backend ComfyUI** أثناء تقارب منصة النماذج: إعداد نوع backend
`"comfyui"` يفشل برسالة `comfyui backend removed`
(`packages/core/src/backends/mod.rs:756-757`). لا يوجد مسار فيديو ذاتي
الاستضافة؛ يمر الفيديو دائمًا عبر backend من نوع `minimax-cloud`.

## دورة حياة المهمة والتسعير

تنتقل مهمة الفيديو عبر `queued → running → done | failed | cancelled`
(`packages/core/src/gateway/video.rs`):

- **الإنشاء (create)** — يُثبَّت صف المهمة (`queued`، تقدم 0) وتُطلق مهمة
  الاستطلاع (poller) (`video.rs:109-176`).
- **قيد التشغيل (running)** — يقدّم المستطلِع المهمة (تقدم 5)، ثم يستطلع كل
  1.5 ثانية، رافعًا التقدم بمقدار 5 كل بضع تكرارات حتى **90**
  (`video.rs:178-275`). تُسجَّل أخطاء الاستطلاع في السجل وتُعاد المحاولة.
- **مكتملة (done)** — تقدم 100، ويُثبَّت عنوان النتيجة والتكلفة المحسوبة،
  ويُسجَّل الاستخدام، ويُبث إشعار `video.done` إلى كل المشتركين
  (`video.rs:332-368`).
- **فاشلة (failed)** — فشل الإرسال أو الاستطلاع → `video.failed`؛ بعد 900
  تكرار استطلاع (نحو 22.5 دقيقة) تفشل المهمة برسالة `generation timed out`.
- **ملغاة (cancelled)** — يضبط `video.cancel` علامة يلاحظها المستطلِع في
  مروره التالي؛ تُعلَّم المهمة `cancelled` ويُطلق `video.failed` مع الخطأ
  `cancelled` (`video.rs:389-400`).

يُسجَّل الاستخدام بالتكلفة الخاصة بالفيديو: يكتب `record_video` سجل استخدام
لكل طلب بصفر tokens وتكلفة دولارية صريحة
(`packages/core/src/billing/mod.rs:496-531`).

**التسعير** خاص بكل نموذج، في جدول `video_pricing`
(`packages/core/src/billing/video.rs`):

| الوضع | الصيغة |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (الافتراضي) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

يرسم `resolution_coeff` مفتاح البكسل للضلع الأقصر (مثل `"768"`) إلى معامل
مضاعف، مع `"*"` كقيمة احتياطية. النماذج التي لا تملك صفًا مهيأً تعود إلى:
الوضع `per_second_resolution`، `base_price` 0.0،
`price_per_second` 0.005، `price_per_frame` 0.0، `resolution_coeff {"*": 1.0}`،
والعملة USD (`billing/video.rs:20-32`). استعلم عن الصفوف عبر
`billing.video.pricing.get` (JWT) وحدّثها بإدراج أو استبدال (upsert) عبر
`billing.video.pricing.set` (token إداري) — انظر
[واجهة JSON-RPC API](../api/jsonrpc.md). وانظر [الفوترة والاستخدام](./billing-usage.md)
لمعرفة كيف تتجمّع سجلات الاستخدام في الفوترة الشهرية.

<!-- src: packages/core/src/gateway/realtime.rs:128-304 (realtime session registry) / packages/core/src/gateway/video.rs:178-387 (video job lifecycle) -->
