---
title: "الأحداث والإشعارات"
description: "المرافق الجانبي لأحداث الخادم (SSE) — chat.stream وmodels.progress وrealtime.event وإشعارات الفيديو."
---

# الأحداث والإشعارات

لا تُسلَّم tokens البث وتقدم النشر وأحداث الوقت الفعلي **على** مقبس WebSocket
الخاص بـ JSON-RPC. ينشئ كل RPC متدفق **معرف جلسة** ويدفع الإشعارات إلى نقطة
نهاية SSE كأحداث مرسلة من الخادم:

```
GET /api/rpc/events?session=<session_id>
```

## وصفة الاشتراك قبل الإرسال

الإشعارات الصادرة بين إعادة استدعاء RPC لمعرف الجلسة وقيام اشتراك SSE
**تُسقَط** (نافذة ما قبل الاشتراك). النمط الموثوق هو:

1. افتح بث SSE أولًا (يحجب حتى يُرفَق معرف جلسة).
2. أطلق RPC الذي يعيد معرف الجلسة (مثل `chat.send`، و`agents.deploy`،
   و`realtime.start`، و`video.create`).
3. اقرأ الإشعارات من بث SSE فور وصولها.

كل إشعار رسالة بنمط JSON-RPC 2.0 تحمل `"jsonrpc": "2.0"`، و`method`
وكائن `params`.

## فهرس الإشعارات

### `chat.stream`

إشعار واحد لكل token، ينتجه `chat.send` (وأي مسار دردشة متدفق يستخدم قناة
جلسة):

```json
{
  "jsonrpc": "2.0",
  "method": "chat.stream",
  "params": { "stream_id": "...", "token": "...", "is_complete": false }
}
```

- `token` — دلتا محتوى واحدة.
- `is_complete` — `false` حتى القطعة الأخيرة (عندما يرفق الـ upstream سبب
  إنهاء، قد تحمل القطعة الأخيرة من المحتوى `is_complete:
  true` أصلًا مع token غير فارغ)؛ ويتبع الإشعار **النهائي** دائمًا بـ `token`
  فارغ و`is_complete: true`.
- يُسلَّم خطأ البث كإشعار نهائي مع `token: "Stream error: ..."` و
  `is_complete: true` (انظر `packages/core/src/gateway/rpc.rs`).

### `models.progress`

تقدم تنزيل النموذج لـ `agents.deploy`، مُمرَّرًا من الوكيل. يأتي `stream_id`
من استجابة `agents.deploy`.

### `realtime.event`

أحداث الخادم لجلسة وقت فعلي مفتوحة ثنائية الاتجاه الكامل، مدفوعة إلى قناة
الجلسة (`packages/core/src/gateway/realtime.rs`). تُمرَّر أحداث العميل
المرسلة عبر RPC `realtime.event` إلى الـ upstream؛ وتصل أحداث الخادم إلى
هنا.

### إشعارات مهام الفيديو

تدفع مهام `video.create` التقدم عبر قناة الجلسة
(`packages/core/src/gateway/video.rs`):

| الأسلوب | الحمولة (params) | المعنى |
| --- | --- | --- |
| `video.progress` | `job_id`, `stream_id`, `status: "running"`, `progress` (0–90) | المهمة قيد التشغيل. |
| `video.done` | `job_id`, `stream_id`, `result`, `cost` | اكتملت المهمة؛ يحمل `result` عنوان الأثر (artifact). |
| `video.failed` | `job_id`, `stream_id`, `error` | فشلت المهمة أو أُلغيت. |

## ملاحظات الترتيب

- بث SSE مرتب لكل جلسة؛ تصل الـ tokens بترتيب التوليد.
- لا يجوز أن يستهلك معرف جلسة واحدًا إلا مشترك SSE واحد؛ افتح البث قبل
  (أو فورًا بعد) RPC الذي يعيد المعرف.
- ترفق ترويسة `x-session-id` على `POST /api/rpc` **استجابة** RPC نفسها بقناة
  جلسة أيضًا — تُستخدم من قبل العملاء الذين يريدون انعكاس الاستجابة على
  البث نفسه.

<!-- src: packages/core/src/gateway/server.rs:192-201 (rpc_events_handler) -->
