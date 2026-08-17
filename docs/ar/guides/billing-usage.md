---
title: "الفوترة والاستخدام"
description: "سجلات الاستخدام، والتكلفة لكل نموذج، والـ billing tiers، وفرض الحصص وحدود المعدل، والمفاتيح المرتبطة بالمشروع، وتسعير الفيديو، وRPC الخاص بـ usage.list."
---

# الفوترة والاستخدام

يقيس arona كل طلب نموذج ويفرض حصصًا وحدود معدل متدرجة عند الـ gateway. تأتي الأسعار لكل نموذج من جدول التسعير المشترك في plana (لا يُعاد تنفيذه أبدًا داخل arona)، وتُخزَّن بيانات الاستخدام كصفوف في `usage_records`، وتُعرض الصورة الشهرية كاملة عبر RPC الخاص بـ `usage.list`.

## سجلات الاستخدام

ينتهي كل طلب مُقاس كصف واحد في جدول `usage_records` (`m20250101_000006_create_usage_records`):

| العمود | النوع | المعنى |
| --- | --- | --- |
| `id` | `UUID` | مفتاح أساسي، مُولَّد تلقائيًا. |
| `api_key_id` | `VARCHAR(64)` | **بادئة المفتاح** — الأحرف الثمانية الأولى من الـ API key (تبدو المفاتيح هكذا `arona-{uuid}`) — أو معرّف اصطناعي `jwt-<user-uuid>` لقنوات RPC المنسوبة إلى JWT. |
| `model` | `VARCHAR(128)` | معرّف النموذج الذي وُجِّه إليه الطلب. |
| `backend` | `VARCHAR(64)` | نوع الـ backend: `gateway`، `rpc`، `realtime`، أو اسم قدرة الـ backend. |
| `prompt_tokens` | `INTEGER` | tokens الإدخال، كما أبلغها الـ upstream أو كما قُدِّرت. |
| `completion_tokens` | `INTEGER` | tokens الإخراج، كما أبلغها الـ upstream أو كما قُدِّرت. |
| `total_tokens` | `INTEGER` | مجموع الاثنين. |
| `cost` | `DOUBLE PRECISION` | التكلفة المحسوبة بالدولار الأمريكي؛ تكون `NULL` عندما لا يوجد صف تسعير للنموذج. |
| `created_at` | `TIMESTAMPTZ` | وقت اكتمال الطلب. |

توجد فهارس (indexes) على `api_key_id` و`model` و`created_at` (الأعمدة التي تفحصها نوافذ التجميع الشهري وحدود المعدل).

## قنوات التسجيل

يُسجَّل الاستخدام على كل قناة مُقاسة:

1. **REST غير المتدفق (non-streaming)** — يسجّل كل من `POST /v1/chat/completions` و`POST /v1/embeddings` الاستخدام المُبلَّغ به بدقة من الـ upstream بمجرد إنتاج الاستجابة.
2. **REST المتدفق (SSE)** — يُعتمد الاستخدام المُبلَّغ من الـ upstream عندما يحمله التدفق (حقل `usage` في القطعة الختامية المتوافقة مع OpenAI)؛ وإلا فتُسجَّل تقديرات الـ tokenizer المحلي المراعي لـ CJK (`estimate_usage`) كما هي. أما التدفقات التي لم تُنتج نصًا ولا استخدامًا فلا تُسجَّل **إطلاقًا**.
3. **RPC الخاص بـ `chat.send`** — يُطبَّق منطق التقدير مقابل الـ upstream نفسه؛ تُنسب الصفوف إلى المعرّف الاصطناعي `jwt-<user-uuid>` حتى تعود وترتبط بالمستخدم.
4. **جلسات الـ realtime** — يسجّل كل نسخ (transcript) مكتمل `response_done` استخدامه من الـ tokens (عندما يكون غير صفري) تحت `jwt-<user-uuid>` مع backend من نوع `realtime`.
5. **مهام الفيديو** — تسجّل المهمة المكتملة تكلفة صريحة بالدولار (انظر [تسعير الفيديو](#video-pricing))؛ وتكون أعداد الـ tokens صفرًا.

التسجيل بأفضل جهد (best-effort): يُسجَّل في السجل أي إدراج فاشل ولا يُفشل الطلب أبدًا.

## حساب التكلفة

تُحسب التكلفة من جدول التسعير المرجعي لكل مليون token (`plana_llm_provider::metering::lookup_pricing`، المشترك بين جميع خدمات celestia-island):

```
cost = prompt_tokens / 1_000_000 * input_price
     + completion_tokens / 1_000_000 * output_price
```

تعتمد مطابقة النموذج في الجدول على البحث الجزئي في معرّف النموذج بأحرف صغيرة (تفوز العائلات الأكثر تحديدًا). عندما لا يوجد صف تسعير لنموذج ما، تكون `cost` بقيمة `NULL`. **لا تعِد تنفيذ التسعير داخل arona — حدِّث جدول plana.**

## مستويات الفوترة

توجد مستويات الفوترة في جدول `billing_tiers`، وتُزرع (seeded) عند أول ترحيل (`m20250101_000007_create_billing_tiers`). عمود الحصة `NULL` يعني «غير محدود» لذلك البُعد. أما المستخدمون الذين لا يملكون `tier_id` فيقعون افتراضيًا على مستوى `free` المزروع.

| المستوى | الحصة الشهرية بالدولار | الحصة الشهرية من الـ tokens | RPM لكل مفتاح |
| --- | --- | --- | --- |
| `free` | $1.00 | 100,000 | 10 |
| `pro` | $20.00 | 5,000,000 | 120 |
| `enterprise` | غير محدود (`NULL`) | غير محدود (`NULL`) | 1000 |

إسناد المستوى عملية إدارية (RPC `billing.plan.set`)؛ ويُعرض المستوى الحالي والاستخدام عبر `billing.plan`.

## فرض الحصص وحدود المعدل

### REST (`/v1/*`)

قبل كل نقطة نهاية REST **مُقاسة** — `/v1/chat/completions` و`/v1/embeddings` و`/v1/video/generations` — يفرض الـ gateway بوابتين (gates) للطلبات الموثَّقة بالمفتاح:

- **الحصة الشهرية** — تُفحص حدود `monthly_quota_usd` و`quota_tokens` الخاصة بالمستوى مقابل الاستخدام المتراكم منذ أول لحظة من الشهر التقويمي الحالي. أي بُعد يبلغ حدّه يُفعِّل البوابة.
- **حد المعدل لكل مفتاح** — يُفحص `rate_limit_rpm` الخاص بالمستوى مقابل عدد الطلبات المسجَّلة لبادئة المفتاح هذه في نافذة الستين ثانية الأخيرة. (`/v1/models` ونقاط نهاية الصحة غير مُقاسة وغير خاضعة للبوابات.)

الرفض هو استجابة HTTP **429 Too Many Requests** مع ترويسة `Retry-After` وجسم خطأ بأسلوب OpenAI:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: <seconds>

{ "error": { "message": "...", "type": "quota_error", "code": "quota_exceeded" } }
```

| الرفض | `type` / `code` | `Retry-After` |
| --- | --- | --- |
| نفاد الحصة الشهرية | `quota_error` / `quota_exceeded` | ثوانٍ حتى بداية **الشهر التقويمي التالي** |
| تجاوز حد معدل المستوى | `rate_limit_error` / `rate_limit_exceeded` | `60` |

### RPC

يمر `chat.send` الموثَّق بـ JWT عبر بوابة الحصة الشهرية نفسها، لكن مقابل نافذة **المستخدم بالكامل** (لا يحمل الاستدعاء أي API key). الرفض هو خطأ JSON-RPC برمز محدد من طرف التطبيق `-32006` (`QUOTA_ERROR`) ونفس رسالة رفض الحصة في REST. لا يوجد حد معدل لكل مفتاح في مسار RPC — فحدود المعدل مرتبطة بالمفتاح واستدعاءات RPC لا تحمل مفتاحًا. أساليب الـ realtime والـ video **عبر RPC** لا تخضع لبوابة الحصص.

## مفاضلة الـ fail-open

الفوترة **بأفضل جهد بحكم التصميم**. إذا فشل استعلام قاعدة البيانات خلف فحص الحصة أو حد المعدل، يُرجع الفحص `Unknown` ويُسمح **بالطلب** (يُسجَّل فقط) بدلًا من حظر الدردشة. يمكن للمشغِّل الاعتماد على استجابات 429 لحماية السعة، لكن لا يجوز التعامل معها كضمان صارم عندما تكون قاعدة البيانات غير سليمة — المفاضلة الموثقة هي توافر مسار الدردشة على حساب القياس الصارم.

## المفاتيح المرتبطة بالمشروع

يمكن إنشاء الـ API keys مع وسم (label) `project` (`api_keys.project`، القيمة `default` عند عدم التعيين). يراعي فرض الحصص هذا الوسم:

- المفتاح الموسوم بمشروع غير `default` يفحص حصته مقابل الاستخدام المنسوب إلى **حوض (bucket) ذلك المشروع نفسه** (`PROJECT_MONTHLY_USAGE_SQL`).
- مفاتيح `default` / غير الموسومة تُبقي نافذة **المستخدم بالكامل**، مطابقةً للسلوك السابق للمشاريع.

صفوف RPC المنسوبة إلى JWT (`jwt-<user-uuid>`) لا تحمل وسم مشروع وتُستبعد **عمدًا** من نوافذ المشروع — لكنها تظل محسوبة ضمن نافذة المستخدم بالكامل، فلا يمكن «إخفاء» مشروع بإرسال حركة المرور عبر قناة RPC.

## تسعير الفيديو

يستخدم توليد الفيديو تسعيرًا خاصًا بالنموذج على نمط المهام (لا معنى لتسعير لكل token في الفيديو). توجد صفوف التسعير في جدول `video_pricing`؛ ويلجأ `compute_cost` إلى افتراضي مؤقت (placeholder) عندما لا يكون أي صف مكوَّنًا.

| الوضع | التكلفة |
| --- | --- |
| `per_call` | `base_price` |
| `per_second_resolution` (الافتراضي) | `base_price + duration_seconds × price_per_second × resolution_coeff[short_side]` |
| `per_frame` | `base_price + frames × price_per_frame` |

`resolution_coeff` كائن JSON مفتاحه قيمة البكسل للجانب الأقصر (مثلًا `"768"`)؛ والمفتاح `"*"` هو الاحتياط. صف التسعير الافتراضي هو الوضع `per_second_resolution`، بقيمة `base_price` 0.0 و`price_per_second` 0.005 و`resolution_coeff {"*": 1.0}`. تُدار الصفوف عبر `billing.video.pricing.get` (أي JWT) و`billing.video.pricing.set` (Bearer `ARONA_ADMIN_TOKEN`)؛ وتُسجَّل التكلفة المحسوبة مقابل مفتاح المهمة عند اكتمالها.

## usage.list

يُرجع `usage.list` (JWT) سجلات الاستخدام المرقّمة (paginated) للمتصل، وتغطي **كلا** النوعين: صفوف الـ API key (مربوطة عبر بادئة المفتاح) وصفوف JWT المنسوبة (مربوطة عبر المعرّف الاصطناعي `jwt-<user-uuid>`)، الأحدث أولًا:

| المعامل | الافتراضي | ملاحظات |
| --- | --- | --- |
| `limit` | `50` | مقيد بين `1..=200`. |
| `offset` | `0` | إزاحة الصفحة. |
| `project` | غير مُعيَّن | عند التعيين، تُرجع السجلات المنسوبة فقط إلى المفاتيح التي تحمل وسم المشروع ذلك (تُستبعد صفوف JWT). |

الاستجابة هي `{ "records": [...], "total", "limit", "offset", "project" }` حيث يحمل كل سجل `id` و`model` و`backend` و`prompt_tokens` و`completion_tokens` و`total_tokens` و`cost` و`created_at`. يستخدم تجميع الحصة الشهرية نفس شكل الربط (join)، لذا يتفق حساب الحصص وعرض الاستخدام دائمًا على النطاق.

## ذات صلة

- [البدء السريع](quickstart.md) — احصل على مفتاح وقدّم أول طلب مُقاس.
- [الإعدادات](configuration.md) — متغيرات البيئة الخاصة بالـ gateway.
- [المصادقة والأمان](auth-security.md) — إنشاء الـ API keys ووسم `project`.
- [Realtime والفيديو](realtime-video.md) — دورة حياة مهمة الفيديو خلف تسعير الفيديو.
- [العمليات](operations.md) — الـ health probes والمراقبة (observability).
- [واجهة REST API المتوافقة مع OpenAI](../api/openai-rest.md) — سطح `/v1/*`.
- [واجهة JSON-RPC API](../api/jsonrpc.md) — `usage.list` و`billing.plan` و`billing.video.pricing.*`.
- [نظرة عامة](README.md)

<!-- src: packages/core/src/billing/mod.rs:452-573 (usage recording & cost), 284-337 (quota/rate-limit gates, fail-open), 421-433 (Retry-After); packages/core/src/migration/mod.rs:233-293 (usage_records schema), 333-339 (tier seed); packages/core/src/gateway/server.rs:492-539 (429 enforcement), 1036-1100/1269-1392 (chat & SSE recording); packages/core/src/gateway/rpc.rs:1075-1175 (usage.list), 1605-1638 (RPC quota gate); packages/core/src/billing/video.rs:20-86 (video pricing) -->

<!-- note: The fact sheet stated that "chat.send, realtime and video RPCs go through the same quota gates". Verified against source: only chat.send is quota-gated on the RPC path (rpc.rs:1605-1638); realtime.start and video.create record usage but apply no quota gate (rpc.rs:1914-1984, 1296-1382). The REST /v1/video/generations endpoint is gated (server.rs:891-895). This page reflects the verified behavior. -->

<!-- note: Realtime sessions additionally record per-response usage under jwt-<user-uuid> (gateway/realtime.rs:61-84) and video jobs record an explicit cost on completion (gateway/video.rs:230-238, 349-356); the fact sheet's three-channel list was extended with these two attribution paths. -->
