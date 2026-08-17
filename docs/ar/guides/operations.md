---
title: "العمليات"
description: "نقاط نهاية الصحة، ومراقبة RUST_LOG، ومهلات الـ upstream، وتعيين الأخطاء، واستكشاف الأخطاء وإصلاحها لخادم arona-server قيد التشغيل."
---

# العمليات

هذه الصفحة مخصصة للمشغِّلين الذين يشغّلون `arona-server serve`. تغطي نقاط نهاية الصحة التي تفحصها، وسطور السجل الجديرة بالبحث (grep)، ونموذج المهلات المطبق على الـ backends في الـ upstream، وكيف تُعيّن أخطاء الـ backends إلى أخطاء HTTP، والمزالق التشغيلية التي تتعثر بها الفرق. أما النشر نفسه فيُغطى في [دليل النشر](./deployment.md).

## مصفوفة الصحة

نقاط نهاية الصحة الثلاث غير موثَّقة وتُرجع `200 OK` كلما كانت العملية تخدم — لا يوجد تمييز بين الجاهزية للعمل (liveness) والاستعداد (readiness):

| نقطة النهاية | الاستجابة |
| --- | --- |
| `/healthz`, `/readyz` | `200` `{"status":"ok","version":<CARGO_PKG_VERSION>,"build_hash":<BUILD_HASH>,"models":<n>,"providers":<n>}` |
| `/v1/health` | نفس الجسم التفصيلي أعلاه |
| `/api/health` | plana `HealthResponse`: `status`، `version` (`CARGO_PKG_VERSION`)، `kind` (`Dev`)، `uptime` (بالثواني)، `network` (transport / region / asn)، `build_hash` (`BUILD_HASH`)، `engine_version` (`"0.1.0"`) |

`/healthz` و`/readyz` اسمان مستعاران لنفس المعالج، ويشاركهما `/v1/health`، لذا فإن الـ probes بأسلوب Kubernetes ومسار الصحة المتوافق مع OpenAI قابلان للتبادل. يضيف `/api/health` مدة التشغيل (uptime) والشبكة وإصدار المحرك. استخدم `/readyz` لموازنات الحمل والمشرفين؛ واستخدم `/api/health` عندما تحتاج إلى الحمولة الأغنى.

## التسجيل

يسجّل الخادم عبر `tracing`، مع تصفية بمتغير `RUST_LOG` القياسي (`RUST_LOG=info` هو الإعداد الشائع؛ و`RUST_LOG=debug` يكشف حركة الـ probes). الأحداث الجديرة بالمعرفة، بترتيب تقريبي حسب التكرار:

| سطر السجل | المستوى | ما يخبرك به |
| --- | --- | --- |
| `chat completions request` / `chat completions SSE request` | info | سطر واحد لكل طلب دردشة، مع `key_prefix` و`model` و`stream` و`request_id` — أبسط مسار تدقيق لكل طلب. |
| `request completed` | info | يسجّله مساعد `logging_middleware` بعد كل استجابة **غير متدفقة** من `/v1/chat/completions` و`/v1/embeddings`: `method` و`path` و`status` و`latency_ms` و`trace_id`. (أما الدردشة المتدفقة فتسجّل `chat completions SSE request` عند البداية بدلًا من ذلك.) |
| `usage recorded` / `usage persisted` | info | سُجِّل صف استخدام (في الذاكرة، مع الـ tokens/التكلفة) ثم كُتب إلى جدول `usage_records`. |
| `external probe: sending` / `external probe: returned` | debug | health probe لنقطة `/v1/models` الخاصة بـ backend خارجي؛ ويقول `matched` ما إذا اكتمل الـ probe ضمن مهلة الـ probe البالغة ثانيتين. |
| `billing gate rejected: monthly quota exceeded` / `billing gate rejected: tier rate limit exceeded` | warn | طلب `/v1/*` رفضته بوابة الفوترة — تلقى العميل 429 بالإضافة إلى `Retry-After`. |
| `rpc billing gate rejected: monthly quota exceeded` | warn | بوابة الحصص في جانب RPC للأساليب الموثَّقة بـ JWT (نافذة المستخدم بالكامل؛ استجابة خطأ JSON-RPC). |
| `restored persisted backends` / `restored backend` / `restored persisted agent nodes` | info | استعادة عند بدء التشغيل: الـ backends المسجَّلة من المدير وعقد الـ agents المحمَّلة من قاعدة البيانات وتُجعل قابلة للتوجيه مرة أخرى. |
| `Shutdown signal received, draining connections…` | info | بدأ الإيقاف الآمن (SIGINT/SIGTERM). |

## نموذج المهلات

تُفرض المهلات على عميل الـ upstream المستخدم للـ backends الخارجية (`packages/core/src/backends/external.rs`):

| المهلة | القيمة | تُطبَّق على |
| --- | --- | --- |
| الاتصال | 10s | إنشاء اتصال TCP/TLS مع الـ upstream. |
| خمول القراءة | 120s لكل قراءة | كل استدعاء upstream؛ كل بايت مستلم يعيد ضبط الساعة، لذا لا يُقطع أبدًا تدفق سليم لكنه بطيء. |
| الإجمالي لغير المتدفق | 600s | استدعاءات الدردشة/التضمينات غير المتدفقة — الـ upstream البطيء لكن الحي لا يمكنه إمساك الطلب إلى الأبد. |
| المتدفق (SSE) | لا شيء | تحمل الاستدعاءات المتدفقة **بدون مهلة إجمالية**؛ التوليدات الطويلة مشروعة ويعتمد كشف التعليق على مهلة خمول القراءة. |
| الـ health probe | 2s | فحص `/v1/models`. |

## تعيين الأخطاء

تُعيّن فشل الـ backends إلى حالات HTTP في معالجات الدردشة/التضمينات (`packages/core/src/gateway/server.rs`):

| الشرط | HTTP | `type` / `code` | الرسالة |
| --- | --- | --- | --- |
| حالة upstream غير 2xx (`UpstreamStatus`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | `upstream <status>: <detail>` |
| فشل نقل upstream (`RequestFailed`) | **502 Bad Gateway** | `server_error` / `bad_gateway` | سلسلة خطأ النقل |
| أي خطأ backend آخر | **500** | `server_error` / `backend_error` | سلسلة الخطأ |
| لا يوجد backend للنموذج (`NoBackend`) | **404** | `invalid_request_error` / `model_not_found` | `No backend available for model: <model>` |
| API key غير صالح (`Unauthorized`) | **401** | `authentication_error` / `invalid_api_key` | `Invalid API key` |
| حد المعدل (`RateLimited`) | **429** | `rate_limit_error` / `rate_limit_exceeded` | `Rate limit exceeded` |

القصد من التصميم: أن يميز المتصلون بين «مزوّدك رفض أو فشل» (502) و«الـ gateway نفسه معطوب» (500). كل جسم خطأ له الشكل نفسه المتوافق مع OpenAI — `{"error":{"message":...,"type":...,"code":...}}` (`json_error_response`). وتحمل استجابات 429 الصادرة عن بوابة الفوترة ترويسة `Retry-After` إضافية وتستخدم `quota_error`/`quota_exceeded` (للحصص) و`rate_limit_error`/`rate_limit_exceeded` (لحد معدل المستوى) على التوالي.

## استكشاف الأخطاء وإصلاحها

### يبقى الـ backend المسجَّل حديثًا fail-closed حتى يُفحص

تبدأ الـ backends الخارجية في حالة صحة غير معروفة وتُبلِّغ عن `"<url> not probed yet"`. وتتحول إلى سليمة عندما (أ) تعمل الجولة الأولى من الـ health checker — فورًا عند بدء التشغيل، ثم كل 60 ثانية — أو (ب) ينجح الـ probe المُطلق والمنسي (fire-and-forget) الذي أُطلق عند التسجيل أو الاستعادة، عادةً خلال ثانية إلى ثانيتين. وحتى ذلك الحين، تفشل الطلبات الموجهة إلى الـ backend بشكل fail-closed بحكم التصميم.

### 404 على `/models` أثناء الـ probe أمر طبيعي لبعض الـ backends

يضرب الـ probe الخارجي `GET {base}/v1/models` (أو `{base}/models` لعناوين أساسية تحمل بادئة مسار). بعض الخوادم المتوافقة مع OpenAI تنفّذ الدردشة لكنها لا تعرض قائمة نماذج — نقطة نهاية خطة البرمجة Zhipu GLM واحدة منها. **يُتسامح مع 404**: يُعلَّم الـ backend كسليم، وتبقى قائمة النماذج المكوَّنة من المدير هي المرجعية للتوجيه. فقط الـ probes الفاشلة فعليًا (مهلة، خطأ شبكة، حالة أخرى غير 2xx) تُعلّم الـ backend كغير سليم.

### التدفقات SSE التي لا تُنتج شيئًا لا تُفوتر

تُسجَّل الاستجابة المتدفقة في الاستخدام فقط عندما أنتج التدفق نصًا **أو** حمل استخدامًا ختاميًا؛ أما التدفق الذي انتهى بلا أي منهما فلا يُسجَّل إطلاقًا. إذا رأيت طلبًا بلا سطر `usage recorded` مطابق، فتحقق مما إذا كان التدفق قد أنتج محتوى فعلًا.

### الإبلاغ عن الإصدار

`version` في أجسام الصحة هو `CARGO_PKG_VERSION`؛ و`build_hash` هو قيمة `BUILD_HASH` الصادرة وقت البناء من `packages/core/build.rs`. قارن `build_hash` عبر العقد للتأكد من أن جميعها تشغّل القطعة (artifact) نفسها.

<!-- note: The /api/health JSON field is `uptime` (u64 seconds), not `uptime_seconds`; the field name comes from plana's HealthResponse (plana packages/protocol-core/src/http.rs:84-92). -->
<!-- src: packages/core/src/gateway/server.rs:1197-1246,1507-1539 -->
