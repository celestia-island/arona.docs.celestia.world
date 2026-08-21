---
title: "الـ Backends"
description: "أنواع الـ backend (external، ollama، engine، minimax-cloud، جسور evernight)، ودلالات الـ URL، وفحص الصحة، واكتشاف النماذج، والـ aliases، والتوجيه."
---

# الـ Backends

الـ **backend** هو جهة علوية (upstream) تخدم حركة النماذج. يوجّه Arona
الطلبات المتوافقة مع OpenAI (`/v1/chat/completions`، `/v1/embeddings`، سرد
النماذج، مهام الفيديو) إلى أحد الـ backends المسجَّلة، ويقيس (meters) كل طلب،
ويُبقي فحص الصحة وجرد النماذج لكل backend محدَّثَين.

يُسجِّل المشرف الـ backends عبر `POST /api/admin/backends` (انظر [واجهة
Admin HTTP API](../api/admin-http.md))، وتُحفظ في جدول `backend_configs`،
وتُستعاد تلقائيًا عند الإقلاع. يحمل كل تسجيل `name` و`type` و`url` و`api_key`
اختيارية وقائمة `models` ثابتة اختيارية. تنجو الـ backends المحفوظة من
إعادة التشغيل؛ تبدأ الـ backends المستعادة بحالة fail-closed وتُفحص فورًا.

## أنواع الـ Backend

| `type` | النقل | البروتوكول | الغرض |
| --- | --- | --- | --- |
| `external` | HTTP(S) | REST متوافق مع OpenAI | أي API للمحادثة/التضمين (سحابي أو مستضاف ذاتيًا) |
| `ollama` | HTTP(S) | API أصلي لـ Ollama (`/api/chat`، `/api/embed`، `/api/tags`) | خادم Ollama محلي أو بعيد؛ يُبنى من الـ URL وحده |
| `engine` | `ws://` / `wss://` | CEP (بروتوكول محرك Celestia)، WebSocket + JSON-RPC | أي محرك يتحدث معيار تبادل CEP (`plana::engine`) |
| `minimax-cloud` | HTTPS | API بنمط مهام MiniMax H3 (إرسال + استطلاع) | توليد الفيديو السحابي |
| `evernight://<node>/<service>` | عنوان جسر (bridge URL) | يُحل عبر وكيل evernight المحلي إلى إعادة توجيه TCP محلية | خدمات صناعية/طرفية لا يمكن الوصول إليها إلا عبر شبكة evernight |
| `agent-{model}` | HTTP | متوافق مع OpenAI (external) | تُسجَّل تلقائيًا عندما ينشر وكيل GPU نموذجًا |

### `external` — أي API HTTP متوافق مع OpenAI

الـ backend للأغراض العامة: إكمال المحادثة (بالدفق وبدونه) والتضمينات ضد
أي خادم يتحدث شكل REST الخاص بـ OpenAI. هيِّئه بعنوان `url` أساسي و`api_key`
(اختيارية) وقائمة `models` ثابتة اختيارية:

```json
{
  "name": "my-gateway",
  "type": "external",
  "url": "https://api.example.com",
  "api_key": "sk-xxx",
  "models": ["my-model-1", "my-model-2"]
}
```

قائمة `models` الثابتة هي المرجع: تُدمَج قبل أي نماذج تُكتشف وقت الفحص
(انظر [Model discovery](#model-discovery)).

### `ollama` — يُبنى من الـ URL وحده

يُبنى backend من نوع Ollama من الـ URL وحده — لا مفتاح API ولا قائمة نماذج.
يتحدث البروتوكولات الأصلية لـ Ollama: `/api/chat` للمحادثة، و`/api/embed`
للتضمينات، و`/api/tags` لفحص الصحة واكتشاف النماذج.

```json
{
  "name": "local-ollama",
  "type": "ollama",
  "url": "http://192.0.2.10:11434"
}
```

### `engine` — CEP عبر WebSocket

يتصل backend من نوع `engine` بمحرك يعرّض `ws://` (أو `wss://`) ويتحدث
**بروتوكول محرك Celestia** (CEP): معيار تبادل WebSocket + JSON-RPC 2.0
معرَّف في `plana::engine`. أي محرك مكتوب بأي لغة ينفّذ تدفق المصافحة →
الطرق → إشعارات الدفق يُسجَّل كـ backend من الدرجة الأولى دون أي كود
محوّل في Arona. طرق الاتصال (wire methods): `Engine.Handshake` (الرسالة
الأولى؛ الهوية + القدرات)، `Engine.Chat`، `Engine.ChatStart` (بالدفق؛ تصل
القطع كإشعارات `Engine.ChatChunk`)، `Engine.Embeddings` و`Engine.Models`.
تُنشأ الاتصالات كسولًا (lazily) عند أول استخدام وتُقطع عند أي خطأ؛ الاستدعاء
التالي يعيد الاتصال والمصافحة.

### `minimax-cloud` — توليد فيديو بنمط المهام

يُشغّل backend الفيديو السحابي API منصة MiniMax H3 Open Platform: أرسل
مهمة توليد، واستطلِع الاكتمال، ثم اقرأ عنوان الـ artifact من النتيجة. هذا
ما حل محل backend ComfyUI المُزال (انظر أدناه)؛ تُرسل مهام الفيديو عبر
`/v1/video/generations` أو طرق RPC الخاصة بـ `video.*` ويتقدّم العمل عبر
إشعارات `video.progress` / `video.done` / `video.failed` (انظر
[الوقت الفعلي والفيديو](realtime-video.md)).

### عناوين الجسر `evernight://`

عنوان backend من الشكل `evernight://<node>/<service>` **لا** يُتَّصل به
مباشرة. يحلّه وكيل evernight على المضيف المحلي (استدعاء JSON-RPC
`Bridge.Connect` عبر نقطة WebSocket الخاصة بالوكيل) إلى إعادة توجيه TCP
محلية، ويعمل الـ backend ضد `http://127.0.0.1:<local_port>` بدلًا من عنوان
بعيد مثبّت في الكود. هذه هي معمارية اللوحة الواحدة: لوحة Arona تصل إلى
الخدمات على العُقد الأخرى (محركات CEP، scepter، ...) عبر الشبكة دون تضمين
عنوان بعيد في أي إعداد.

مهمة إبقاء الاتصال (keepalive) تفحص النفق كل 15 ثانية؛ عندما يُعاد تشغيل
الطرف البعيد ويُعاد تأسيس النفق على منفذ محلي جديد، **يُعاد بناء** الـ backend
المتأثر بشفافية بالعنوان الجديد — يحتفظ الإعداد المحفوظ بعنوان `evernight://`
حتى تعيد عمليات إعادة التشغيل حلّه. بالنسبة إلى backends من نوع `engine`،
يُكيَّف إعادة التوجيه المُحلَّلة `http://127.0.0.1:<port>` إلى `ws://`
لنقل WebSocket.

### النماذج المنشورة بواسطة الـ agent تُسجَّل تلقائيًا

عندما يُنهي وكيل GPU نشر نموذج، يُسجِّل الـ gateway backend باسم
`agent-{model_id}` (وهو `ExternalApiBackend` عبر `http://{agent host}:{port}`)
فيصبح النموذج قابلًا للتوجيه فورًا؛ إيقاف النشر يلغي تسجيله مجددًا. انظر
[مجموعة الـ Agent](agent-cluster.md) لدورة حياة النشر الكاملة.

### `comfyui` مرفوض

يُرفض نوع backend `comfyui` صراحةً مع الخطأ `comfyui backend removed`:
أُزيل backend ComfyUI أثناء التقارب نحو منصة النماذج، ويعمل توليد الفيديو
الآن عبر `minimax-cloud`. تسجيل backend من نوع `comfyui` يُعيد HTTP 400.

## دلالات الـ URL

الطريقة التي يُحدَّد بها عنوان أساسي مُهيَّأ لنقاط النهاية الفعلية تتقرر حسب
ما إذا كان للـ URL مكوّن مسار:

- **أساس بنمط الجذر (Root-style)** — يُعامَل الـ URL الذي يكون مساره فارغًا
  أو `/` كجذر مضيف ويحافظ على اصطلاح OpenAI `/v1`: `{base}/v1/chat/completions`،
  `{base}/v1/models`. أمثلة: `http://192.0.2.20:8429`،
  `https://api.deepseek.com`.
- **أساس بنمط المسار (Path-style)** — يُعامَل الـ URL ذو المسار غير الفارغ
  كبادئة API كاملة يخدمها الخادم فعلًا، ويُلحق نقطة النهاية مباشرة:
  `{base}/chat/completions`، `{base}/models`. هذا ما تحتاجه الخوادم
  المتوافقة مع OpenAI خارج اصطلاح `/v1`. خطة برمجة Zhipu GLM هي المثال
  الأساسي: واجهتها تعيش عند
  `https://open.bigmodel.cn/api/coding/paas/v4` مع المحادثة مباشرة عند
  `{base}/chat/completions` و**لا نقطة `/models` إطلاقًا** — جذر
  `/api/paas/v4` القياسي يعيد أخطاء رصيد لمفاتيح خطة البرمجة.
- **شرطة مائلة زائدة (trailing slash)** على الـ URL الأساسي المُهيَّأ
  تُطبَّع بعيدًا حتى لا يُنتج الالتحام شرطة مائلة مزدوجة.

## الفحص والصحة

يفحص فاحص صحة في الخلفية كل backend مسجَّل كل **60 ثانية**؛ تُجلب قائمة
الـ backends من جديد في كل جولة، لذا تُلتقط الـ backends المسجَّلة بعد
الإقلاع دون إعادة تشغيل. كل تسجيل إداري يطلق أيضًا فحصًا فوريًا حتى ينقلب
الـ backend إلى حالة صحية خلال ~1–2 ثانية بدلًا من انتظار جولة الفاحص
التالية.

- **الـ backends الخارجية** تفحص `GET {base}/models` (أو `{base}/v1/models`
  للأساسات بنمط الجذر) مع **مهلة ثانيتين**. **يُتسامَح مع 404**: بعض
  الخوادم تنفّذ المحادثة دون إظهار قائمة نماذج (خطة برمجة GLM لا تملك نقطة
  `/models`)، لذا يضع 404 الـ backend في حالة صحية وتصبح قائمة `models`
  المُهيَّأة من المشرف مصدر التوجيه. المهلات وفشل الشبكة واستجابات أخرى غير
  2xx تضع الـ backend في حالة غير صحية.
- **Backends من نوع Ollama** تفحص `/api/tags` بالمهلة نفسها (ثانيتان).
- تبدأ الـ backends بحالة **fail-closed** — تُبلَّغ كـ `not probed yet` —
  حتى أول فحص ناجح، لذا لا يستقبل backend مسجَّل حديثًا (أو مستعاد) أي
  حركة قبل التحقق منه.

تُخزَّن حالة الصحة مؤقتًا لكل backend ويستشيرها الموجّه في كل طلب؛ تُستبعَد
الـ backends غير الصحية من اختيار المرشحين (انظر [Routing](#routing)).

## اكتشاف النماذج

يُعلن الـ backend عن معرّفات النماذج التي يخدمها، ويطابق الموجّه الطلبات
ضد ذلك الإعلان:

- الـ backends **الخارجية** تُعلن النماذج المُستخرجة من استجابة الفحص
  (يُقبل كل من مصفوفة `data` ومصفوفة `models`)، مدمجة مع القائمة الثابتة
  المُهيَّأة من المشرف — تحتفظ المعرّفات الثابتة بالترتيب والأولوية،
  وتُزال تكرارات المعرّفات الديناميكية وتُلحق. عندما لا يملك الخادم نقطة
  نماذج، تكون القائمة الثابتة وحدها مصدر التوجيه.
- **Ollama** تُعلن الوسوم (tags) المعادة من `/api/tags`.
- النماذج **المنشورة بواسطة الـ agent** تُعلن `model_id` المنشور بالضبط.

السطح العام هو `GET /v1/models` (مُصادَق عليه)، الذي يسرد النماذج
القابلة للتوجيه لكل backend صحي (انظر [REST API المتوافق مع
OpenAI](../api/openai-rest.md)).

## الـ Aliases وتطبيع الأسماء

تُحدد الـ aliases معرّف نموذج مطلوبًا إلى معرّف هدف. يُحلّ الـ alias أولًا
في التوجيه، لذا يُخدَم طلب الـ alias بأي backend يُعلن الهدف:

```json
{ "alias": "fast-chat", "target": "deepseek-v4-flash" }
```

تُدار الـ aliases عبر نقاط نهاية المشرف `/api/admin/aliases` وتدخل حيز
التنفيذ فورًا.

تطبّع مطابقة الأسماء أيضًا وسوم نمط Ollama: backend يسرد
`nomic-embed-text:latest` يطابق طلبًا مجردًا لـ`nomic-embed-text`، فتُحل
طلبات التضمين/المحادثة دون تدبير لاحقة `:latest`. الوسم الصريح
(`qwen3:0.6b`) يطابق ذلك الوسم بالضبط فقط.

## التوجيه

كل طلب يُحل عبر الموجّه، الذي يختار backend واحدًا:

1. **حل الـ alias** — يُحدَّد معرّف النموذج المطلوب عبر جدول الـ aliases
   (إن وُجد).
2. **تلميح الـ provider** — حقل `provider` اختياري يرشّح المرشحين حسب اسم
   الـ backend (أو اسم النوع، مثل `cloud` لـ backends الفيديو).
3. **المرشحون الأصحاء فقط** — يجب أن يبلّغ الـ backend عن `Healthy` *و*
   يجتاز قاطع الدائرة (circuit breaker) الخاص به (3 إخفاقات متتالية تفتح
   القاطع لمدة 30 ثانية، مع استدعاء اختبار واحد نصف-مفتوح) ليكون قابلاً
   للاختيار.
4. **اختيار الأقل عدًّا (Least-count)** — تُرتَّب المرشحات التي تخدم
   النموذج حسب عداد الطلبات الخاص بكل backend ويُختار الأقل حمولة. هذا
   يوزّع الحمل على الـ backends الصحية التي تخدم النموذج نفسه.
5. **ارتباط الجلسة (Session affinity)** — عندما يحمل الطلب
   `conversation_id`، تُثبَّت المحادثة على الـ backend الذي استخدمته أول
   مرة. يعيش التثبيت في خريطة مراجع `Weak`، لذا يختفي backend مُزال من
   الخريطة دون انحراف فهارس. الارتباط بأفضل جهد: إعادة استخدام الـ backend
   نفسه عبر أدوار المحادثة تتيح للجهة العلوية إعادة استخدام حالة وقت
   التشغيل الخاصة بالمحادثة (سياقات دافئة، مخازن KV). إذا أصبح الـ backend
   المثبَّت غير صحي (أو مات التثبيت مع backend مُزال)، يتراجع الموجّه إلى
   اختيار جديد بأقل عدًّا **ويعيد ربط** المحادثة.

إذا لم يخدم أي backend صحي النموذج، يفشل التوجيه: نموذج مجهول يُنتج
`model not found` (HTTP 404)، ونموذج معروف لكن غير قابل للوصول يُنتج
`all backends unhealthy`، الذي يُظهر كخطأ خادم داخلي 500. يُحجز HTTP 502
للفشل المبلَّغ عنه من upstream *قابل للوصول* (استجابات upstream غير 2xx
وفشل النقل بعد الاختيار). انظر [العمليات](operations.md) لخريطة الأخطاء
الكاملة.

<!-- src: packages/core/src/backends/mod.rs:699-771 (build_backend_from_config) / packages/core/src/backends/external.rs:49-60 (join_api_path) / packages/core/src/routing/mod.rs:24-26,102-200 (model matching + routing) -->
<!-- note: `all backends unhealthy` surfaces as HTTP 500, not 502: AllUnhealthy maps to BackendError::Unavailable, and backend_error_to_http (packages/core/src/gateway/server.rs:1197-1206) reserves 502 for UpstreamStatus/RequestFailed only. -->
<!-- note: ollama embeddings use `/api/embed`, not `/api/embeddings` (packages/core/src/backends/ollama.rs:314-320); probe is `/api/tags` (ollama.rs:53-92). -->
