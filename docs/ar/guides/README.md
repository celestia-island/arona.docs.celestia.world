---
title: "Arona"
description: "منصة النشر الذاتي والإدارة عن بُعد لنماذج الذكاء الاصطناعي — gateway، backends، billing، agents، memory."
---

# Arona

**منصة النشر الذاتي والإدارة عن بُعد لنماذج الذكاء الاصطناعي.**

أرونا منصة **backend خالص** مكتوبة بلغة Rust (axum): إنها gateway لنماذج الذكاء
الاصطناعي متوافقة مع OpenAI *وأيضًا* مستوى إدارة للنماذج التي تشغّلها على عتادك
الخاص. توفّر واجهة REST API المتوافقة مع OpenAI عبر `/v1/*`، ومستوى إدارة
JSON-RPC 2.0 (`/api/rpc`)، ومستوى التحكم في الوكلاء (`/ws/agent`)، وواجهة
Swagger UI على `/docs`.

لا توجد **لوحة تحكم ويب مدمجة ولا CLI دردشة مدمج** — واجهة الدردشة + الإدارة
تعيش في [shittim-chest](https://github.com/celestia-island/shittim-chest)،
الذي يتواصل مع أرونا عبر سطح RPC. تركز أرونا على الجانب الخادمي: التوجيه
(routing)، والفوترة (billing)، والمصادقة (auth)، ونشر النماذج، والوكلاء
(agents)، والذاكرة (memory).

## مصفوفة الميزات

| المنطقة | ما توفّره أرونا |
| --- | --- |
| **جوهر المحادثة** | `chat.completions` متوافق مع OpenAI (بالبثّ وبدونه)، وقوائم `embeddings` و`models`؛ بثّ مع مقطع `[DONE]` نهائي و usage حقيقي في المقطع الأخير. |
| **الـ backends** | upstreams مسجّلة إداريًا: `external` (أي HTTP API متوافق مع OpenAI)، و`ollama`، و`engine` CEP (WebSocket)، وفيديو `minimax-cloud`، وعناوين جسر `evernight://` إلى الخدمات الصناعية/الطرفية (edge). |
| **المصادقة** | أزواج JWT للوصول/التحديث (15 دقيقة / 7 أيام)، ومفاتيح API بصيغة `arona-{uuid}` مخزّنة كـ SHA-256 hashes، وثلاث مستويات إدارية، وسياسة كلمات مرور، وتحديد معدل ثنائي المسار (dual-track rate limiting). |
| **الفوترة والاستخدام** | مستويات مبدئية (free / pro / enterprise)، وسجلات usage لكل طلب على كل قناة، وجدول أسعار plana، وتحديد حصص لكل مشروع، و429 + `Retry-After`. |
| **إدارة النماذج** | تنزيل النماذج (مصادر `hf:` / `ms:` / `gh:`)، ونشر عُقد GPU عبر `_agent`، والتسجيل التلقائي للنماذج المنشورة كـ backends قابلة للتوجيه. |
| **Realtime والمتعددة الوسائط** | جلسات `realtime.*` ثنائية الاتجاه كاملة، وقناة الإدراك/التحكم `engine.invoke`، ومهام توليد فيديو غير متزامنة (MiniMax cloud). |
| **مجموعة الوكلاء** | تتصل عُقد GPU عبر `/ws/agent`، وتوزيع على الأقل تحميلًا (least-loaded placement)، وارتباط الجلسات (session affinity)، واستمرارية العُقد عبر إعادة التشغيل. |
| **بوابة الذاكرة** | ذاكرة طويلة المدى عبر entelecheia Philia: حقن الاسترجاع (recall injection)، وكتابة الحلقات (writeback episodes)، والتدهور الصريح (explicit degradation). |
| **العمليات** | فحوصات الصحة (health probes)، وتتبّع `RUST_LOG`، وربط أخطاء الـ upstream (502 مقابل 500)، وإيقاف تشغيل سلس (graceful shutdown)، وترحيل تلقائي عند الإقلاع. |

## التموضع

أرونا **gateway + منصة**: توجّه حركة مرور النماذج إلى الـ backends الخاصة بك،
وتنشر النماذج على وكلاء GPU لديك، وتقيس كل شيء.

- مقابل **pi** — pi هو مساعد CLI يتحدث إلى النماذج؛ لا يملك أرونا دردشة CLI.
  أرونا هي المنصة التي يتحدث *إليها* pi (والأدوات الأخرى).
- مقابل **one-api / new-api** — هاتان بوابتان لمفاتيح API أمام مزوّدي النماذج؛
  تضيف أرونا **نشر النماذج** (تنزيل الأوزان وتشغيلها على وكلائك)، ومستوى إدارة
  RPC كاملًا، ومستويات فوترة (billing tiers)، والذاكرة.
- مقابل **LiteLLM** — gateway نظيرة؛ تملك أرونا بالإضافة إلى ذلك دورة حياة نشر
  النماذج خلف الـ gateway.

## ابدأ من هنا

- [البدء السريع](quickstart.md) — من البداية إلى النهاية مع الـ mock upstream المدمج.
- [الإعدادات](configuration.md) — كل متغيرات البيئة.
- [النشر](deployment.md) — التثبيت على العتاد المباشر، systemd، Docker، والإشراف.
- [الـ backends](backends.md) — أنواع الـ backends، ودلالات الـ URL، والفحص (probing).
- [واجهة REST API المتوافقة مع OpenAI](../api/openai-rest.md) — مسارات `/v1/*`.
- [واجهة JSON-RPC API](../api/jsonrpc.md) — مستوى الإدارة الكامل.

## هيكل المستودع

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

تمت إزالة لوحة التحكم على الويب من هذا المستودع، وهي تعيش الآن في
[shittim-chest](https://github.com/celestia-island/shittim-chest) (chest #291).

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
