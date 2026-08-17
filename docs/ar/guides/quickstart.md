---
title: "البدء السريع"
description: "جولة شاملة من البداية إلى النهاية في أرونا مع الـ mock upstream المدمج: الترحيل (migrate)، والتشغيل (serve)، وتسجيل backend، وإنشاء API key، والدردشة."
---

# البدء السريع

يرشدك هذا الدليل خلال إعداد شامل من البداية إلى النهاية لأرونا على جهاز واحد
باستخدام **الـ mock upstream المدمج** — لا حاجة لأوزان نماذج حقيقية، أو GPU،
أو حساب API خارجي. بحلول النهاية سيكون لديك:

- gateway أرونا قيد التشغيل (واجهة REST API المتوافقة مع OpenAI عبر `/v1/*`
  بالإضافة إلى مستوى إدارة JSON-RPC عبر `/api/rpc`)،
- الـ mock upstream مسجّلًا كـ backend من نوع `external`،
- حساب مستخدم و API key،
- جولة دردشة عاملة بدون بثّ **وبالبثّ** مع الـ mock،
- سجلات usage ظاهرة عبر `usage.list`.

## المتطلبات الأساسية

- **سلسلة أدوات Rust** (انظر `rust-toolchain.toml` في جذر المستودع).
- **Python 3** مع `aiohttp` — مطلوبة فقط للـ mock upstream
  (`pip install aiohttp`).
- نسخة **PostgreSQL قيد التشغيل** وعنوان الاتصال (connection URL) الخاص بها.

## 1. اضبط البيئة

يقرأ أرونا إعداده من متغيرات البيئة **عند إقلاع العملية**. اثنان إلزاميان:
`DATABASE_URL` و`JWT_SECRET` — يرفض الخادم الإقلاع بدونهما (إلا إذا كان
`MOCK_MODE=1`). `ARONA_ADMIN_TOKEN` موصى به بشدة: بدونه، يعيد كل مسار
`/api/admin/*` رمز 401.

```bash
export DATABASE_URL="postgres://arona:CHANGE_ME@127.0.0.1:5432/arona"
export JWT_SECRET="CHANGE_ME-a-long-random-string"
export ARONA_ADMIN_TOKEN="CHANGE_ME-an-admin-token"

# Optional: open self-service sign-up (truthy values: 1, true, yes, on).
# The first registered user becomes the admin either way.
export ARONA_REGISTRATION_OPEN=1
```

تُقرأ هذه المتغيرات مرة واحدة عند إقلاع العملية — إذا غيّرتها، أعد تشغيل
الخادم. راجع [الإعدادات](configuration.md) للمرجع الكامل للمتغيرات.

## 2. نفّذ الترحيل وابدأ الخادم

```bash
cargo run -p _cli -- migrate    # run database migrations explicitly
cargo run -p _cli -- serve      # start the gateway
```

`serve` وحده كافٍ لقاعدة بيانات جديدة: فهو يهاجر تلقائيًا عند الإقلاع.
يربط الخادم `0.0.0.0:8420` افتراضيًا (يمكن تجاوزه عبر `ARONA_HOST` /
`ARONA_PORT`).

## 3. ابدأ الـ mock upstream

في طرفية ثانية:

```bash
python3 scripts/mock/server.py
```

الـ mock هو خادم aiohttp يستمع على `127.0.0.1:8429` افتراضيًا
(`ARONA_MOCK_PORT` يتجاوز المنفذ). يطبع مفتاح API الخاص به عند الإقلاع،
ويوفّر أيضًا `GET /api/test-key` الذي يعيد `{"api_key": ..., "base_url": ...}`.
يعرض عددًا من معرّفات النماذج — بما فيها `gpt-5.5` المستخدم أدناه — ويجيب
على إكمالات الدردشة العادية والمتدفقة معًا.

التقط المفتاح المطبوع:

```bash
export MOCK_KEY="<the API key printed on mock startup>"
```

## 4. سجّل الـ mock كـ backend خارجي

تُسجَّل الـ backends عبر واجهة HTTP الإدارية:

```bash
curl -X POST http://127.0.0.1:8420/api/admin/backends \
  -H "Authorization: Bearer $ARONA_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "url": "http://127.0.0.1:8429",
    "api_key": "'"$MOCK_KEY"'",
    "name": "mock",
    "models": ["gpt-5.5"]
  }'
```

يُفحص الـ backend فور التسجيل ويصبح سليمًا (healthy) خلال ~1-2 ثانية؛ وإلى أن
يكتمل ذلك الفحص يبقى في حالة "لم يُفحص بعد" مغلقة عند الفشل (fail-closed)
(انظر صندوق استكشاف الأخطاء أدناه). يُحفظ الإعداد، لذا يبقى الـ backend
قائمًا بعد إعادة التشغيل.

## 5. سجّل حسابًا وسجّل الدخول

تعيش الحسابات على مستوى JSON-RPC عبر `POST /api/rpc`. ولأن `ARONA_REGISTRATION_OPEN=1`
مضبوط، فإن `auth.register` مفتوح؛ أول مستخدم مُسجَّل يصبح المدير.

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1, "method": "auth.register",
    "params": {
      "email": "dev@example.com",
      "password": "Test-password1",
      "name": "Dev"
    }
  }'
```

يجب أن تكون كلمات المرور 8 أحرف على الأقل **وتحتوي** على 3 من فئات الأحرف
الأربع على الأقل (أحرف كبيرة، أحرف صغيرة، أرقام، رموز خاصة). ثم سجّل الدخول
للحصول على زوج JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "auth.login",
    "params": {"email": "dev@example.com", "password": "Test-password1"}
  }'
```

صدّر `access_token` من الاستجابة:

```bash
export JWT="<access_token from the login response>"
```

## 6. أنشئ API key

`keys.create` مصادَق عبر JWT ويعيد السر **الكامل** `arona-{uuid}` مرة واحدة
بالضبط — قاعدة البيانات تخزّن فقط SHA-256 hash له، لذا انسخه الآن:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 3, "method": "keys.create", "params": {"name": "dev"}}'
```

```bash
export AR_KEY="<the arona-... secret returned by keys.create>"
```

## 7. الدردشة (بدون بثّ)

```bash
curl -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}]}'
```

ستستلم كائن إكمال (completion) بأسلوب OpenAI مع `choices[0].message`
وكتلة `usage`.

## 8. الدردشة (بالبثّ)

نفس النقطة الطرفية مع `"stream": true` تستجيب بأحداث يرسلها الخادم
(server-sent events): مقطع `data:` واحد لكل token، وينتهي بمقطع
`data: [DONE]` نهائي:

```bash
curl -N -X POST http://127.0.0.1:8420/v1/chat/completions \
  -H "Authorization: Bearer $AR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.5", "messages": [{"role": "user", "content": "Hello!"}], "stream": true}'
```

## 9. تحقق من الاستخدام

يسجّل كل جولة دردشة صفّ usage تحت بادئة المفتاح. استعلم عنه بالـ JWT:

```bash
curl -X POST http://127.0.0.1:8420/api/rpc \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 4, "method": "usage.list", "params": {}}'
```

يجب أن ترى سجلًا أو أكثر لطلبات `gpt-5.5` التي أجريتها أعلاه.

## استكشاف الأخطاء وإصلاحها

- **`No backend available for model: gpt-5.5` (HTTP 404، `code: model_not_found`)** —
  لا يوجد backend مسجّل يخدم معرّف النموذج هذا. إما أن الـ backend لم يُسجَّل
  أبدًا (أو أن قائمة `models` الخاصة به لا تتضمن المعرّف)، أو أن استدعاء
  التسجيل فشل. تحقق عبر `GET /api/admin/backends` (باستخدام admin token).
- **`all backends unhealthy` (HTTP 500، `backend_error`)** — الـ backend *مسجَّل*
  للنموذج لكن لا يوجد مرشّح سليم. يبدأ الـ backend الخارجي المُسجَّل حديثًا في
  حالة "لم يُفحص بعد" مغلقة عند الفشل، ويصبح سليمًا بمجرد اكتمال فحص التسجيل،
  بعد ~1-2 ثانية؛ إذا دارت دردشة داخل تلك النافذة فستصادف هذا الخطأ. أعد
  المحاولة بعد لحظة، أو تحقق من أن الـ mock يعمل فعلًا على `127.0.0.1:8429`.
- **HTTP 401 على `/v1/*`** — غياب ترويسة `Authorization` يولّد
  `Missing Authorization header. Use: Bearer <api_key>`؛ والمفتاح غير المعروف
  يولّد `Invalid API key`. أعد التحقق من `$AR_KEY` (السر الكامل، وليس البادئة).
- **HTTP 401 `Admin access required` على `/api/admin/*`** — token الحامل
  (bearer) لا يطابق `ARONA_ADMIN_TOKEN`، أو أن المتغير غير مضبوط (حينها يرفض
  المسار دائمًا). أعد تشغيل الخادم بعد ضبطه.
- **`auth.register` يفشل برسالة "Registration is closed"** — التسجيل معطّل عندما
  لا يكون `ARONA_REGISTRATION_OPEN` بقيمة صادقة (truthy). اضبط
  `ARONA_REGISTRATION_OPEN=1` **قبل** بدء الخادم (يُقرأ عند الإقلاع)، أو كن
  أول مستخدم — أول مستخدم مُسجَّل مسموح له دائمًا ويصبح المدير.
- **حدود المعدل HTTP 429** — ثلاثة حدود مستقلة يمكن أن تتفعّل:
  - الحد في الذاكرة لكل مفتاح، 60 RPM افتراضيًا
    (`ARONA_API_RATE_LIMIT_RPM`) → `Rate limit exceeded. Try again later.`;
  - حد 10 RPM لكل مفتاح في مستوى الفوترة المجاني (free) → 429 مع ترويسة
    `Retry-After: 60`;
  - حصة المستوى المجاني الشهرية البالغة $1 / 100k-token → 429 مع `Retry-After`
    يشير إلى فترة الفوترة التالية.

## الخطوات التالية

- [الإعدادات](configuration.md) — كل متغيرات البيئة.
- [الـ backends](backends.md) — أنواع الـ backends، ودلالات الـ URL، والفحص (probing).
- [النشر](deployment.md) — التثبيت على العتاد المباشر، systemd، Docker.
- [واجهة REST API المتوافقة مع OpenAI](../api/openai-rest.md) — كامل سطح `/v1/*`.
- [واجهة JSON-RPC API](../api/jsonrpc.md) — مستوى الإدارة المستخدم أعلاه.

<!-- src: packages/core/src/gateway/server.rs (chat + admin routes), packages/core/src/gateway/run.rs:40-50 (startup checks), scripts/mock/server.py (mock upstream) -->
<!-- note: vs the source fact sheet — (1) the register-time probe that flips a new backend healthy lives in server.rs:688-693, not run.rs:122-127 (those lines cover restored backends after restart); (2) during the fail-closed probe window routing returns 500 "all backends unhealthy" (routing/mod.rs:183,340; gateway/mod.rs:144-146), while the 404 model_not_found fires only when no registered backend serves the model; (3) the user-facing auth.register error when registration is closed is "Registration is closed" (gateway/rpc.rs:424), the underlying AuthError is "registration is disabled" (auth.rs:712-713). -->
