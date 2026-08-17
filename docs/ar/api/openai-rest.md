---
title: "واجهة REST API المتوافقة مع OpenAI"
description: "مرجع /v1/* بأسلوب OpenAI — إكمال المحادثة، التضمينات، قائمة النماذج، توليد الفيديو غير المتزامن، أشكال الأخطاء وحدود المعدل."
---

# واجهة REST API المتوافقة مع OpenAI

يُعرّض arona سطح REST متوافقًا مع OpenAI تحت `/v1/*` لدردشة LLM، والتضمينات، وسرد النماذج، وفحص الصحة (health probing)، وتوليد الفيديو غير المتزامن. أي SDK من OpenAI موجّه إلى عنوان القاعدة (base URL) يعمل للدردشة والتضمينات؛ وتتبع نقاط الفيديو الطرفية اصطلاح OpenAI القائم على نمط الإرسال/الاستطلاع (submit/poll).

جميع أجسام الطلبات والاستجابات بصيغة JSON. تستخدم الأخطاء شكلًا موحّدًا (انظر [Errors](#errors))؛ وفشل المصادقة على مستوى طبقة الوسيطة (middleware) هو الاستثناء الوحيد ويُعاد كنص عادي (انظر [Authentication](#authentication)).

## نظرة سريعة على النقاط الطرفية

| الطريقة | المسار | الوصف |
| --- | --- | --- |
| `POST` | `/v1/chat/completions` | جولة دردشة، مع البث أو بدونه. |
| `POST` | `/v1/embeddings` | متجهات التضمين لمدخل واحد أو عدة مدخلات. |
| `GET` | `/v1/models` | نماذج الموجّه (router) مدمجة مع نماذج البدء السريع. |
| `GET` | `/v1/health` | `{"status": "ok", "version", "build_hash", "models", "providers"}`. |
| `POST` | `/v1/video/generations` | إرسال مهمة توليد فيديو غير متزامنة. |
| `GET` | `/v1/video/generations/{id}` | استطلاع حالة/نتيجة مهمة فيديو. |

تُعد `/api/health` و `/healthz` و `/readyz` فحوص جاهزية إضافية (أسماء بديلة بأسلوب Kubernetes لـ `/v1/health`).

## المصادقة

تتحقق نقاط الدردشة والتضمينات والفيديو الطرفية من الهوية باستخدام **API key** في ترويسة `Authorization: Bearer`. تُنشأ API keys عبر مستوى الإدارة (`keys.create`، انظر [واجهة JSON-RPC API](./jsonrpc.md#keys)) وتكون بالشكل `arona-<uuid>`. وتُخزَّن من جهة الخادم (server-side) كتجزئات SHA-256.

```
Authorization: Bearer arona-CHANGE_ME
```

- **ترويسة مفقودة** → `401` كنص عادي: `Missing Authorization header. Use: Bearer <api_key>`.
- **مفتاح غير صالح أو مُلغى** → `401` كنص عادي: `Invalid API key`.
- يقبل `GET /v1/models` إضافةً إلى ذلك **JWT** access token (يصدره `auth.login` / `auth.register`) حتى تتمكن لوحة التحكم على الويب من سرد النماذج باستخدام نفس token الذي تستخدمه لمستوى RPC. وبالنسبة إلى هذه النقطة الطرفية تكون الرسائل `Missing Authorization header. Use: Bearer <api_key_or_jwt>` و `Invalid API key or JWT`.

رفض مستوى الوسيطة (middleware) يكون على شكل أجسام نصية عادية، وليس شكل الخطأ JSON الموضح في [Errors](#errors) — لا يُنتَج شكل JSON إلا عندما يصل الطلب إلى معالج (handler).

يمر كل طلب `/v1` مصادَق عليه أيضًا عبر **محدِّد معدل لكل مفتاح في الذاكرة (in-memory per-key rate limiter)** (الافتراضي 60 RPM، ونافذة 60 ثانية، وقابل للتهيئة عبر `ARONA_API_RATE_LIMIT_RPM`). تجاوزه يُرجع `429` كنص عادي: `Rate limit exceeded. Try again later.` وتُفرض حصص وحدود معدل مستوى الطبقة (tier) بشكل منفصل وتُعيد أخطاء 429 بصيغة JSON مع ترويسة `Retry-After` (انظر [429 and Retry-After](#429-and-retry-after)).

> تُغطى إدارة API keys والمشاريع ونطاقاتها في
> [المصادقة والأمان](../guides/auth-security.md).

## POST /v1/chat/completions

نقطة الطرفية الأساسية للدردشة المتوافقة مع OpenAI، مع دعم البث وامتدادات خاصة بـ arona (`conversation_id`، `memory`، `extra`، `provider`).

### جسم الطلب

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `model` | string | نعم | معرّف النموذج كما يسرده `GET /v1/models`. |
| `messages` | array | نعم | جولات الدردشة، انظر أدناه. |
| `stream` | boolean | لا | الافتراضي `false`. عندما يكون `true` تكون الاستجابة دفق SSE (انظر [Streaming](#streaming)). |
| `temperature` | number | لا | درجة حرارة أخذ العينات (sampling temperature)، تُمرَّر إلى المنبع (upstream). |
| `max_tokens` | integer | لا | سقف tokens الإكمال، يُمرَّر إلى المنبع (upstream). |
| `conversation_id` | string | لا | الارتباط بالجلسة (session affinity) + الاستمرارية (persistence). يجب أن توجد المحادثة وأن تخص مستخدم API key (وإلا `403` `conversation_forbidden`، أو `404` `conversation_not_found` إذا لم تكن موجودة). تُحفَظ جولة المستخدم وقت الإرسال ورد المساعد عند اكتمال الجولة؛ ويثبّت التوجيه (routing) المحادثة على backend الذي خدمها أولًا. |
| `memory` | boolean | لا | تجاوز بوابة الذاكرة (memory gateway). الافتراضي `true` (يُحقن استدعاء الذاكرة عندما تكون بوابة الذاكرة مفعّلة)؛ و`false` يعطّل حقن الاستدعاء لهذا الطلب. |
| `extra` | object | لا | تمرير حر الشكل يُدمج في المستوى الأعلى من حمولة المنبع (انظر أدناه). |
| `tools` | array | لا | تعريفات استدعاء الدوال بأسلوب OpenAI، تُمرَّر حرفيًا إلى المنبع. |
| `provider` | string | لا | تلميح اختيار backend صريح يطابق **اسم** backend (أو نوعه) دون حساسية لحالة الأحرف. عند ضبطه، لا تكون سوى الـ backends المطابقة للتلميح مرشحة. |

**مدخلات `messages`** هي `{ "role": "user" | "assistant" | "system", "content": "..." }`. ويُمرَّر امتدادان إلى المنبع لأحمال العمل متعددة الوسائط / الوكلاء (multimodal / agent):

- `images` — الصور المرفقة لطلبات الرؤية (مصفوفة من كائنات `{ "media_type", "data", "position" }`؛ يعرضها backend الخارجي كأجزاء محتوى OpenAI من نوع `image_url`).
- `tool_calls` — حمولات استدعاء الدوال التي ينتجها نموذج المنبع، لتُعاد في الجولات اللاحقة. يجب على backend الخارجي تمريرها وإلا فقدت خطوط أنابيب الوكلاء (agent pipelines) (مثل سلاسل مهارات scepter) كل استدعاء أداة.

**قواعد دمج `extra`**: يُدمج كل مفتاح من `extra` في حمولة طلب المنبع على المستوى الأعلى، مع ضمانين صارمين — المفاتيح المحجوزة `model`، `messages`، `stream`، `temperature`، `max_tokens` و`options` **لا** تُستبدل أبدًا، ولا يُستبدل أيضًا أي مفتاح سبق أن ضبطه gateway نفسه. وتُتجاهل قيم `extra` غير الكائنية.

**مدخلات `tools`** هي `{ "type": "function", "function": { "name",
"description"?, "parameters"? } }` وتُمرَّر حرفيًا.

### الاستجابة غير المتدفقة

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1720000000,
  "model": "Qwen/Qwen3-1.7B",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!" },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 12, "completion_tokens": 2, "total_tokens": 14 }
}
```

- قد يحمل `choices[].message` حقل `tool_calls` لجولات استدعاء الدوال.
- تُعاد حالة ذاكرة الطلب في ترويسة الاستجابة **`X-Arona-Memory`**: `enabled` | `disabled` | `offline`.

### البث

اضبط `"stream": true`. تكون الاستجابة دفق SSE من نوع `text/event-stream` — سطر `data:` واحد لكل جزء (chunk)، يحمل كل منها JSON واحدًا من نوع `ChatChunk`:

```json
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1720000000,"model":"Qwen/Qwen3-1.7B","choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"},"finish_reason":null}]}
```

- يحمل `choices[].delta` حقل `content` (وفروق `tool_calls` مع `index`/`id`/`type`/`function` لدفقات استدعاء الدوال).
- في المنبع المتوافق مع OpenAI، يحمل **الجزء النهائي** حقل `usage` بأعداد tokens الفعلية؛ يسجّله gateway (مع اللجوء إلى تقدير tokenizer محلي للمنبع الذي لا يبلّغ عن الاستخدام أبدًا، مثل ollama / ws_engine).
- ينتهي الدفق بـ `data: [DONE]`.
- يُسلَّم خطأ الدفق كحدث `data:` يحمل `{"error": {"message": "...", "type": "server_error", "code": "stream_error"}}`؛ ويأتي حدث `[DONE]` بعده على أي حال، ويُتخطَّى تسجيل الاستخدام وحفظ رد المساعد للدفق الفاشل.
- ترويسة `X-Arona-Memory` موجودة على استجابة SSE أيضًا.

### مثال

```bash
curl http://192.0.2.10:8080/v1/chat/completions \
  -H "Authorization: Bearer arona-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-1.7B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

## POST /v1/embeddings

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `model` | string | نعم | معرّف نموذج التضمين (مثل `nomic-embed-text` — الاسم المجرد يطابق أيضًا وسمًا `:latest`). |
| `input` | string or string[] | نعم | مدخل واحد، أو عدة مدخلات. |

الاستجابة: `{ "object": "list", "data": [ { "object": "embedding", "embedding": [...], "index": 0 } ], "model": "...", "usage": { "prompt_tokens", "completion_tokens", "total_tokens" } }`.

## GET /v1/models

يسرد النماذج القابلة للتوجيه اليوم: قائمة نماذج كل backend مسجَّل وسليم، مدمجة مع **نماذج البدء السريع** المدمجة (تُعلَن دائمًا، حتى قبل تسجيل أي backend): `Qwen/Qwen3-0.6B`، `Qwen/Qwen3-1.7B`، `HuggingFaceTB/SmolLM2-1.7B-Instruct`، `google/gemma-3-1b-it`، `microsoft/Phi-4-mini-instruct`، `deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B`.

```json
{
  "object": "list",
  "data": [
    { "id": "Qwen/Qwen3-0.6B", "object": "model", "owned_by": "huggingface" }
  ]
}
```

تظهر نماذج البدء السريع مع ضبط `owned_by` على مزوّدها (provider)؛ وتحمل نماذج الموجّه اسم الـ backend المالك.

## توليد الفيديو

نقاط طرفية للفيديو بأسلوب المهام (task-style) للـ backends القادرة على الفيديو (مثل `minimax-cloud`، انظر [الـ backends](../guides/backends.md)). تتقدّم المهام بشكل غير متزامن؛ استطلع نقطة الحالة حتى `done`.

### POST /v1/video/generations

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `model` | string | نعم | معرّف نموذج فيديو مسجَّل على backend قادر على الفيديو. |
| `prompt` | string | نعم | مطالبة التوليد (prompt). |
| `negative_prompt` | string | لا | المطالبة السلبية (negative prompt). |
| `images` | array | لا | صور التكييف/المرجعية كمصفوفة من كائنات `{ "data_base64": "...", "mime_type": "image/png" }`. |
| `duration_seconds` | integer | لا | المدة المطلوبة. |
| `width` / `height` | integer | لا | دقة المخرجات. |
| `provider` | string | لا | تلميح اختيار backend صريح (اسم backend). |
| `extra` | object | لا | تجاوزات سير عمل خاصة بـ backend (seed، steps، cfg، ...). |

عند النجاح → `200`:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "queued",
  "created_at": 1720000000
}
```

الأخطاء: `400` `missing_fields` عند غياب `model` أو `prompt`؛ و`503` `video_backend_error` / `no_backend` عندما لا يخدم أي backend سليم قادر على الفيديو النموذج؛ و`429` `quota_error` / `quota_exceeded` عند استنفاد الحصة الشهرية.

### GET /v1/video/generations/{id}

يعيد حالة المهمة:

```json
{
  "id": "3f2d...-uuid",
  "object": "video.generation",
  "model": "minimax-h3",
  "status": "running",
  "progress": 40,
  "result": null,
  "error": null,
  "cost": 0.0,
  "created_at": 1720000000
}
```

- `status`: `queued` | `running` | `done` | `failed` | `cancelled`؛ يتقدّم `progress` من 0 إلى 90 أثناء التشغيل ويصل إلى 100 عند `done`.
- `result` (عند `done`): `{ "url", "duration_seconds"?, "width"?, "height"?,
  "format"? }` — يشير `url` إلى الأثر المولَّد الذي يخدمه backend.
- يُعبَّأ `error` (عند `failed` / `cancelled`) و`cost` عند الاقتضاء.
- الأخطاء: `400` `bad_id` لمعرّف غير UUID؛ و`404` `no_job` عندما لا توجد المهمة أو تعود إلى API key آخر.

تبعث مهام الفيديو أيضًا التقدّم عبر الـ sidecar SSE الخاص بـ RPC (`video.progress` / `video.done` / `video.failed`، انظر [الأحداث والإشعارات](./events.md#video-job-notifications)).

## الأخطاء

تستخدم أخطاء مستوى gateway شكلًا واحدًا (`json_error_response`):

```json
{
  "error": { "message": "...", "type": "...", "code": "..." }
}
```

| الحالة | `type` / `code` | متى |
| --- | --- | --- |
| `400` | `invalid_request` / `missing_fields`, `missing_index`, `bad_id`, ... | حقول طلب مشوّهة أو مفقودة. |
| `403` | `auth_error` / `conversation_forbidden` | `conversation_id` يعود إلى مستخدم آخر. |
| `404` | `invalid_request_error` / `model_not_found` | لا يوجد backend يخدم النموذج المطلوب. الرسالة: `No backend available for model: <model>`. |
| `404` | `invalid_request` / `conversation_not_found` | المحادثة غير موجودة. |
| `404` | `not_found` / `no_job` | مهمة الفيديو غير موجودة. |
| `502` | `server_error` / `bad_gateway` | منبع غير 2xx: الرسالة `upstream <status>: <detail>` (التفاصيل من جسم خطأ المنبع، محدودة بـ 4 كيلوبايت). كما تُرسم إخفاقات النقل (connect/read/timeout) إلى 502 مع سلسلة الخطأ. |
| `500` | `server_error` / `backend_error` | إخفاقات backend أخرى (مثل عدم دعم backend للعملية). |
| `500` | `server_error` / `internal_error` | أي خطأ داخلي متبقٍ في gateway. |
| `429` | انظر أدناه | رفض الحصص / حدود المعدل مع `Retry-After`. |

## 429 و Retry-After

تتضمن استجابات 429 ترويسة `Retry-After` (بالثواني) حتى يتراجع عملاء OpenAI المتوافقون:

| المحفّز | جسم الحالة | `Retry-After` |
| --- | --- | --- |
| تجاوز الحصة الشهرية | `{"error":{"message":"Monthly quota exceeded for your billing tier. ...","type":"quota_error","code":"quota_exceeded"}}` | ثوانٍ حتى الشهر التالي. |
| حد المعدل لكل دقيقة لمستوى الطبقة (tier) | `{"error":{"message":"Rate limit exceeded for your billing tier. Retry later.","type":"rate_limit_error","code":"rate_limit_exceeded"}}` | `60`. |
| محدِّد لكل مفتاح في الذاكرة (الافتراضي 60 RPM) | نص عادي `Rate limit exceeded. Try again later.` | لا شيء (رفض من الوسيطة). |

تُوصف الطبقات (tiers) ونطاق الحصص ومحاسبة الاستخدام في [الفوترة والاستخدام](../guides/billing-usage.md).

## تسجيل الاستخدام

يسجّل كل طلب `/v1` صف استخدام تحت بادئة API key (`arona-XX`) عند اكتماله (دردشة غير متدفقة، دردشة متدفقة عند الجزء النهائي، تضمينات، ومهام الفيديو عند اكتمالها مع تكلفتها المحسوبة). راجع [الفوترة والاستخدام](../guides/billing-usage.md) لنموذج التسجيل وكيفية فرض الحصة.

<!-- src: packages/core/src/gateway/server.rs:128-163 (route table), 997-1123 (chat_completions), 1248-1423 (chat SSE), 1425-1503 (models/embeddings), 876-993 (video), 449-539 (errors/429); packages/core/src/backends/mod.rs:17-35 (embeddings), 146-221 (ChatMessage/ChatRequest), 432-484 (ChatResponse/ChatChunk) -->

<!-- note: Fact-sheet deltas confirmed against the code: (1) middleware auth rejections are plain text, not the JSON error shape (middleware.rs:29,56-77); (2) GET /v1/models auth messages read `Bearer <api_key_or_jwt>` / `Invalid API key or JWT` (middleware.rs:135-142); (3) video `images` is an array of {data_base64, mime_type} objects (backends/mod.rs:44-50), and GET status values are queued/running/done/failed/cancelled (gateway/video.rs:94-101); (4) the 404 `model_not_found` error `type` is `invalid_request_error` (server.rs:1212-1217); (5) `provider` mismatch surfaces as 500 `backend_error` via RouterError::ProviderNotFound (gateway/mod.rs:147-149). -->
