---
title: "الاختبار"
description: "هرم اختبارات أرونا — اختبارات الوحدات، والتكامل المحكم، والتكامل المقيد بـ PostgreSQL، واختبارات الدخان على خادم حي، وخادم الـ mock، وانضباط اختبار الدخان بالبيانات الاعتمادية الحقيقية."
---

# الاختبار

تُرتَّب اختبارات أرونا في طبقات بحيث يكون تشغيل `cargo test` الافتراضي
سريعًا ومحكمًا (hermetic) ولا يحتاج قاعدة بيانات ولا شبكة، بينما تكون
المجموعات الأثقل اشتراكات صريحة (opt-in) تمارس سطح الاتصال الحقيقي
وPostgreSQL حقيقية. ترسم هذه الصفحة الطبقات، والأوامر التي تشغّلها،
وانضباط مساحة العمل حول تشغيلات الدخان بالبيانات الاعتمادية الحقيقية.

## اختبارات الوحدات

معظم التغطية اختبارات وحدات عادية داخل `packages/core/src`:
217 دالة `#[test]` / `#[tokio::test]`، إضافة إلى نحو 23 أخرى عبر
`packages/agent` و`packages/cli`. تعمل مع:

```bash
cargo test --workspace
```

لا شبكة، لا قاعدة بيانات. المجموعات الرئيسية:

- **auth.rs** — سياسة كلمة المرور (≥8 أحرف و≥3 من 4 فئات الأحرف)،
  وتحويلات `::uuid` في SQL الخام INSERT/REVOKE، والافتراضيات للطلبات،
  وقراءات علامة المسؤول التي تعود إلى `false`.
- **billing/mod.rs** — حساب الحصة على بُعد التكلفة *أو* الـ tokens، والنافذة
  الشهرية (`month_start`، `seconds_until_month_end`)، وسقف حد المعدل
  (لا يُفعَّل إلا *عند* RPM، `None` = بلا حدود)، وحرّاس شكل SQL لاستعلامات
  الاستخدام الشهري / المستوى / نافذة المفتاح، و`estimate_usage` الذي يفضّل
  الأرقام المبلَّغ عنها من الـ upstream.
- **routing/mod.rs** — حل الاسم المستعار، ومطابقة لاحقة `:latest`، وتلميحات
  الـ provider، واختيار الأقل تحميلًا، وتثبيت المحادثة.
- **gateway/mod.rs** — تسجيل backends الوكلاء: تسجيل `agent-{model_id}`،
  وإعادة التسجيل التي تستبدل (لا تكرر)، وإلغاء التسجيل الذي يعيد الموجّه
  إلى حالته.

## التكامل المحكم (يعمل دائمًا، بلا قاعدة بيانات)

يحتوي `packages/core/tests/gateway_integration.rs` على ثلاثة اختبارات تعمل
دائمًا تمارس منطق التسلسل/التعاقد الحقيقي دون لمس قاعدة بيانات:

- **A1** — تعاقد تسلسل صدى المعرف في JSON-RPC: معرفات الطلبات الرقمية
  والنصية والخالية تعبر جولة كاملة عبر enum `Id` الخاص بـ plana مع الحفاظ
  على النوع.
- **A2** — تعاقد رموز خطأ بوابة المسؤول: يبقى `AUTH_ERROR` (-32005، مجهول
  الهوية) و`ADMIN_REQUIRED` (-32007، مصادَق غير مسؤول) متمايزين، ويعيشان في
  النطاق المحدد بالتنفيذ، ولا يصطدمان أبدًا بtokens plana أو برمز الفوترة
  `QUOTA_ERROR` (-32006).
- **A3** — `estimate_usage`: يفوز الاستخدام المبلَّغ عنه من الـ upstream
  حرفيًا؛ وبدونه ينتج تقدير الـ tokenizer المحلي عدّادات prompt/completion
  غير صفرية مجموعها هو حاصل جمعهما.

يضيف `packages/core/tests/smoke.rs` ثلاثة اختبارات أخرى تعمل دائمًا: كشف
العتاد، والمسار الجذري لسجل النماذج، والافتراضيات الإعدادية تحت
`MOCK_MODE=1`.

## التكامل المقيد بـ PostgreSQL

المجموعة الكاملة للـ gateway داخل العملية — `packages/core/tests/gateway_integration.rs`
— تشغّل موجّه axum الكامل على منفذ حلقة محلية (loopback) عشوائي، وتسجّل
upstreams وهمية (mock) متوافقة مع OpenAI قابلة للتخلص عبر واجهة الإدارة
الحقيقية، وتقود سطح الاتصال بـ reqwest. ولأن `AuthManager` يتحدث إلى
PostgreSQL في كل مسار (حتى `MOCK_MODE=1` يزرع الحسابات *داخل قاعدة
البيانات* فقط)، تُقيَّد هذه المجموعة خلف `ARONA_TEST_PG=1` وتُتخطى
افتراضيًا. الاختبارات العشرة:

- **T1** تسجيل + دخول + `keys.create`/`keys.list` (المفتاح الخام مقنّع في
  القوائم، بادئة `arona-`).
- **T2** دردشة REST مع تثبيت سجل الاستخدام في PostgreSQL.
- **T3** صدى معرف JSON-RPC عبر الاتصال (مسارا النجاح والخطأ).
- **T4** بوابة المسؤول على `agents.list`: مجهول الهوية → `AUTH_ERROR`،
  غير مسؤول → `ADMIN_REQUIRED`.
- **T5** upstream يعيد 401 → HTTP 502 `bad_gateway` مع تفاصيل الـ upstream.
- **T6** استطلاع وقت التسجيل ينشر النماذج (يظهر النموذج في `GET /v1/models`
  خلال 10 ثوانٍ دون قائمة نماذج ثابتة).
- **T7** استمرار المحادثة عبر `chat.send` (يستقر كلا الدورين في
  `conversations.get`).
- **T8** بوابة فوترة المستوى المجاني: 10 RPM لكل مفتاح، والطلب الحادي عشر
  في النافذة يعيد 429 `rate_limit_exceeded`.
- **T9** بث SSE مع تسجيل الاستخدام النهائي من الـ upstream.
- **T10** JSON تالف → 400؛ نموذج مجهول → 404 `model_not_found`.

شغّله بسطر PostgreSQL القابل للتخلص من توثيق الوحدة
(gateway_integration.rs:18-26):

```bash
docker run -d --name arona-it-pg-$$ \
  -e POSTGRES_PASSWORD=it_pw -e POSTGRES_USER=it -e POSTGRES_DB=it \
  -p 127.0.0.1::5432 postgres:15-alpine
# read the mapped host port, then:
DATABASE_URL=postgres://it:it_pw@127.0.0.1:<port>/it \
  ARONA_TEST_PG=1 cargo test -p _core --test gateway_integration -- --ignored
```

هذه بيانات اعتماد مثال لحاوية الاختبار القابلة للتخلص فقط — لا توجهها أبدًا
إلى قاعدة بيانات حقيقية.

## اختبار الدخان على خادم حي

يسير `packages/core/tests/auth_flow.rs` عبر السلسلة الكاملة
`register → login → keys.create → /v1/models → /v1/chat/completions →
usage.list` ضد خادم أرونا **حي**، عاكسًا حلقة المصادقة المنشورة. وهو
`#[ignore]` افتراضيًا — لا يلمس تشغيل `cargo test` العادي الشبكة أبدًا.
شغّله صراحةً:

```bash
ARONA_TEST_RUN=1 cargo test -p _core --test auth_flow -- --ignored
```

المقابض (Knobs):

- `ARONA_TEST_URL` — عنوان الأساس (الافتراضي `http://127.0.0.1:8420`).
- `ARONA_TEST_EXPECT_CHAT=1` — تأكيد صارم أن `POST /v1/chat/completions`
  يعيد 200. بدونه يؤكد الاختبار نجاح المصادقة فقط (لا 401/403)، لأن بيئة
  الهدف قد لا يكون فيها provider استدلال مهيأ.

تتضمن المجموعة أيضًا اختبارات سالبة: يجب رفض إكمال الدردشة غير المصادَق
و`GET /v1/models` غير المصادَق معًا برمز 401.

## خادم الـ Mock

`scripts/mock/server.py` مزيف (fake) متوافق مع OpenAI مبني على aiohttp
يستخدمه البدء السريع وتشغيلات الدخان. يخدم `POST /v1/chat/completions`
(غير المتدفق وSSE)، و`GET /v1/models`، و`GET /api/health`، وسطح JSON-RPC
WebSocket/HTTP عند `/api/rpc`، ومرافقًا جانبيًا SSE عند `/api/rpc/events`،
و`GET /api/test-key` الذي يعيد مفتاح الـ mock API كي تستطيع الخدمات الأخرى
اكتشافه. يستمع على المنفذ 8429 افتراضيًا (يتجاوزه `ARONA_MOCK_PORT`،
والمضيف بـ `ARONA_MOCK_HOST`). يستخدمه [البدء السريع](quickstart.md) لإقامة
بيئة شاملة من طرف إلى طرف دون مزودي نماذج حقيقيين.

## انضباط اختبار الدخان بالبيانات الاعتمادية الحقيقية

تشغيلات الدخان ضد مزودين حقيقيين (DeepSeek / GLM) **ليست** اختبارات مستودع
عمدًا — تتطلب بيانات اعتماد حقيقية وأموالًا حقيقية، فلا يمكنها العيش في CI
أو في شجرة git. اصطلاح مساحة العمل، الموثق في توثيق وحدة gateway_integration
(gateway_integration.rs:54-55)، هو:

- تعيش ملفات الإثبات تحت `/mnt/work/arona-pr*-smoke.md` — محلية في مساحة
  العمل، ولا تُرسل إلى git أبدًا.
- تأتي البيانات الاعتمادية من البيئة فقط؛ وتُبقى الميزانيات صغيرة.
- كل PR يمس مسار الاستدلال يحصل على سجل إثبات مكتوب.

خادم الـ mock هو البديل لهذه التشغيلات في CI والتطوير المحلي؛ أما الدخان
بالبيانات الاعتمادية الحقيقية فخطوة بشرية وقت الإصدار.

## CI

يشغّل `.github/workflows/ci.yml` كلاً من `cargo fmt` و`cargo clippy` و`cargo test
--workspace` و`cargo-deny` على عدّاءات المنظمة ذاتية الاستضافة
(`[self-hosted, linux, x64, local]`)؛ ويعكس `ci-hosted.yml` الفحوص نفسها
على عدّاءات مستضافة على GitHub. يبني `.github/workflows/docs.yml` موقع
التوثيق هذا بـ lagrange وينشره إلى GitHub Pages عند الدفعات التي تمس
`docs/**`.

<!-- src: packages/core/tests/gateway_integration.rs:1-55 (module docs, A1-A3, T1-T10); packages/core/tests/auth_flow.rs:1-29 (live knobs); packages/core/tests/smoke.rs:1-25; scripts/mock/server.py:1-33 (mock config); .github/workflows/ci.yml, docs.yml -->
