---
title: "واجهة HTTP API للإدارة"
description: "سطح إداري يعمل بـ Bearer token — تسجيل/سرد/إزالة الـ backends وإدارة أسماء النماذج البديلة (aliases) عبر /api/admin/*."
---

# واجهة HTTP API للإدارة

يدير سطح `/api/admin/*` **الـ backends** الخاصة بـ gateway (مزوّدي النماذج في المنبع) و**الأسماء البديلة (aliases)** (إعادة توجيه اسم النموذج → معرّف النموذج). وهو النظير HTTP لمستوى إدارة RPC (انظر [واجهة JSON-RPC API](./jsonrpc.md)) ويُستخدم أساسًا من قبل المشغّلين وواجهة الإدارة.

## المصادقة

يتطلب كل مسار `/api/admin/*` ما يلي:

```
Authorization: Bearer $ARONA_ADMIN_TOKEN
```

يُقرأ `ARONA_ADMIN_TOKEN` من البيئة عند بدء العملية (`GatewayServer::new`). إذا كان المتغير **غير مضبوط**، أو لم يطابق token المقدَّم، يُرفض الطلب بـ `401`:

```json
{
  "error": {
    "message": "Admin access required",
    "type": "auth_error",
    "code": "unauthorized"
  }
}
```

تُطابَق بادئة الحامل دون حساسية لحالة الأحرف (`Bearer` أو `bearer`).

> على عكس سطح `/v1/*`، لا تتراجع مصادقة الإدارة أبدًا
> إلى API keys أو JWTs، ويُفرضها مقارنة token دقيقة —
> بدّل token بإعادة تشغيل العملية بقيمة جديدة.

## الـ Backends

الـ backends هي المنبع القابل للتوجيه خلف gateway. يجعل التسجيل backend قابلًا للتوجيه فورًا، ويحفظ إعداده لاستعادته عند إعادة التشغيل، ويفحصه (ينقلب إلى سليم خلال ~1–2 ثانية) وبالنسبة لعناوين الجسر (bridge) يبقي النفق حيًا. تُفصَّل أنواع الـ backends ودلالات URL في [الـ backends](../guides/backends.md).

### POST /api/admin/backends — تسجيل backend

جسم الطلب (جميع الحقول اختيارية إلا حيث يُشار خلاف ذلك):

| الحقل | النوع | ملاحظات |
| --- | --- | --- |
| `type` | string | نوع backend. أحد: `external` (أي واجهة HTTP API متوافقة مع OpenAI)، `ollama` (خادم ollama محلي أو بعيد)، `engine` (محرك CEP عبر `ws://`/`wss://`)، `minimax-cloud` (واجهة فيديو سحابية). تُحل أسماء محرك MDD (`llama_cpp`، `vllm`، `ollama`، `cloud`، `external_api`، `candle`، `native`، ...) عبر المخطط (planner). يُرفض `comfyui` (**`comfyui backend removed`**)؛ وأي شيء آخر → `400` `unknown_type`. الافتراضي `ollama` عند غيابه. |
| `url` | string | عنوان URL الأساسي لـ backend. تُحل عناوين الجسر `evernight://<node>/<service>` عبر وكيل evernight المحلي إلى إعادة توجيه TCP محلية (فشل الحل → `502` `evernight_unreachable`). الافتراضي `http://localhost:11434`. |
| `api_key` | string | API key اختياري للمنبع، يُرسَل كـ `Authorization: Bearer` في استدعاءات المنبع. |
| `name` | string | اسم backend. الافتراضي قيمة `type` عند غيابه. يُستخدم كتلميح `provider` للتوجيه وكهوية صف الإعداد. |
| `models` | string[] | قائمة نماذج ثابتة. مصدر التوجيه عندما لا يكتشف الفحص أيًا منها. بالنسبة إلى backends من نوع `external`، تُدمج النماذج المكتشفة بعد القائمة الثابتة (تحتفظ المعرفات الثابتة بالأولوية)؛ وتعرض backends من نوع `engine` مخبأ النماذج المكتشف أولًا ثم تلحق المعرفات الثابتة بعده؛ ولا يجري `minimax-cloud` أي اكتشاف نماذج (فحصه يرسل فحوصات صحة إلى `/v1/query/available_models` فقط) ويخدم القائمة الثابتة وحدها. يتجاهله `ollama`، الذي يكتشف النماذج من `/api/tags`. |
| `workflow` | object | اختياري. قديم (legacy) — كان يستهلكه سابقًا backend ComfyUI المحذوف؛ لا يقرؤه أي backend حالي (أُبقي لتوافق عمود `backend_configs`). |

مثال:

```bash
curl -X POST http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "name": "mock-upstream",
    "url": "http://192.0.2.20:11434",
    "api_key": "sk-xxx",
    "models": ["Qwen/Qwen3-1.7B"]
  }'
```

عند النجاح → `200`:

```json
{ "status": "ok", "message": "backend registered" }
```

الآثار الجانبية للتسجيل:

- يُسجَّل backend ويكون **قابلًا للتوجيه فورًا** (دون حاجة لإعادة تشغيل).
- يُحفظ الإعداد في جدول `backend_configs` ويُستعاد عند بدء التشغيل (يُسجَّل إخفاق قاعدة البيانات في السجل لكنه لا يحجب الاستجابة أبدًا).
- يعمل **فحص** من نوع أطلق وانسَ (fire-and-forget) فورًا بحيث ينقلب backend إلى سليم خلال ~1–2 ثانية بدل بقائه مغلقًا عند الإخفاق حتى جولة مدقق الصحة التالية بعد 60 ثانية.
- بالنسبة إلى عناوين `evernight://`، تراقب **مهمة إبقاء الحياة (keepalive)** النفق: عند إعادة الاتصال بمنفذ محلي جديد تعيد بناء backend وتسجيله بشفافية تحت نفس الاسم.

### GET /api/admin/backends — سرد الـ backends

```json
{
  "backends": {
    "count": 2,
    "health": [
      ["backend_0:ExternalApi", "Healthy"],
      ["backend_1:Ollama", { "Unhealthy": "connection refused" }]
    ]
  },
  "models": [
    { "id": "Qwen/Qwen3-1.7B", "object": "model", "owned_by": "mock-upstream" }
  ]
}
```

- `backends.count` — عدد الـ backends **السليمة**.
- `backends.health` — تسمية `backend_<index>:<kind>` لكل backend وحالة الصحة (`Healthy` / `Degraded` / `Unhealthy`). `<index>` هو فهرس تسجيل الموجّه المستخدم بواسطة `DELETE /api/admin/backends`.
- `models` — كل معرّف نموذج قابل للتوجيه اليوم (نفس قائمة `GET /v1/models`، دون دمج البدء السريع؛ انظر [REST المتوافقة مع OpenAI](./openai-rest.md#get-v1models)).

### DELETE /api/admin/backends — إزالة backend

يُعرَّف بواسطة **فهرس** الموجّه في جسم JSON — وليس بالاسم:

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/backends \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "index": 0 }'
```

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `index` | integer | نعم | فهرس تسجيل الموجّه، مطابقًا لتسمية `backend_<index>` في تقرير الصحة الخاص بـ `GET /api/admin/backends`. |

- غياب `index` → `400` `{"error":{"message":"Missing 'index' field","type":"invalid_request","code":"missing_index"}}`.
- فهرس خارج النطاق → `404` `{"error":{"message":"Backend not found at given index","type":"invalid_request","code":"not_found"}}`.
- عند النجاح → `200` `{ "status": "ok", "message": "backend removed" }`.
- يُحذف صف `backend_configs` المحفوظ بأقصى جهد ممكن: يُستعاد اسم backend من `owned_by` لقائمة نماذجه؛ ويُبقي عدم التطابق الصف في المخزن (تُسجَّل إخفاقات قاعدة البيانات ولا تكون قاتلة أبدًا).

## الأسماء البديلة (aliases)

ترسم الأسماء البديلة (aliases) اسم نموذج إلى آخر (`alias` → `target`) بحيث تُوجَّه طلبات معرّف نموذج واحد إلى نموذج backend مختلف. تُحل الأسماء البديلة قبل التوجيه، لذا تنطبق بشكل موحد على عمليات البحث في الدردشة والتضمينات والفيديو.

> الأسماء البديلة هي **حالة توجيه في الذاكرة فقط** —
> لا تُحفظ وتضيع عند إعادة التشغيل. سجّلها بعد بدء التشغيل
> أو أعد إنشاءها من حالة التجهيز الخاصة بك.

### POST /api/admin/aliases — إضافة اسم بديل

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `alias` | string | نعم | اسم النموذج الذي سيطلبه العملاء. |
| `target` | string | نعم | معرّف النموذج الذي تُوجَّه إليه الطلبات. |

```bash
curl -X POST http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }'
```

- غياب `alias` → `400` `missing_alias`؛ غياب `target` → `400` `missing_target`.
- عند النجاح → `200` `{ "status": "ok", "message": "alias added" }`.
- إضافة اسم بديل موجود يستبدل هدفه.

### GET /api/admin/aliases — سرد الأسماء البديلة

```json
{
  "aliases": [
    { "alias": "my-model", "target": "Qwen/Qwen3-1.7B" }
  ]
}
```

تُعاد الأزواج مرتبة حسب الاسم البديل.

### DELETE /api/admin/aliases — إزالة اسم بديل

| الحقل | النوع | مطلوب | ملاحظات |
| --- | --- | --- | --- |
| `alias` | string | نعم | الاسم البديل المراد إزالته. |

```bash
curl -X DELETE http://192.0.2.10:8080/api/admin/aliases \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{ "alias": "my-model" }'
```

- غياب `alias` → `400` `missing_alias`.
- إزالة اسم بديل غير معروف نجاح بلا تأثير → `200` `{ "status": "ok", "message": "alias removed" }`.

## ملخص الاستمرارية

| المورد | محفوظ؟ | الاستعادة عند إعادة التشغيل |
| --- | --- | --- |
| الـ Backends | نعم — جدول `backend_configs` (مفتاح `name`، إدراج أو تحديث عند التسجيل، حذف عند الإزالة). | نعم: تُستعاد عند بدء التشغيل؛ تبدأ backends من نوع external مغلقة عند الإخفاق وتنقلب سليمة بعد جولة الفحص الأولى. تُعاد حل عناوين `evernight://` عبر الجسر عند بدء التشغيل. |
| الأسماء البديلة (aliases) | لا — في الذاكرة `Router.aliases` فقط. | لا. |

<!-- src: packages/core/src/gateway/server.rs:577-600 (check_admin), 588-698 (add_backend), 700-722 (list_backends_admin), 724-775 (remove_backend), 779-819 (add_alias_handler), 821-838 (list_aliases), 840-869 (remove_alias_handler); packages/core/src/backends/mod.rs:675-771 (BackendConfig / build_backend_from_config); packages/core/src/routing/mod.rs:276-292 (aliases); packages/core/src/gateway/run.rs:87-132 (backend restore) -->

<!-- note: Fact-sheet delta: DELETE /api/admin/backends identifies the backend by a JSON-body `index` field (router registration index), not by name — server.rs:737-747. Also confirmed: aliases are in-memory only (routing/mod.rs:276-292); `workflow` is legacy and ignored by all current backends (backends/mod.rs:682-688); `models` is ignored by the `ollama` kind, which discovers models from `/api/tags` (backends/mod.rs:738-739, ollama.rs:57-91). -->
