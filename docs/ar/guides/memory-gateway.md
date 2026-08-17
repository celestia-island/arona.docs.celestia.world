---
title: "Memory Gateway"
description: "ذاكرة طويلة المدى للدردشة — حقن الاسترجاع، والكتابة العائدة للحلقات، والتحكم لكل طلب، وحالات الترويسة، وRPC الخاص بـ memory.status / memory.delete."
---

# Memory Gateway

يمنح Memory Gateway أدوار الدردشة إمكانية الوصول إلى **ذاكرة طويلة المدى** مخزَّنة في خدمة الذاكرة scepter / Philia الخاصة بـ entelecheia. في كل دور دردشة، يستعلم arona الخدمة عن ذكريات ذات صلة بالمحادثة، ويحقنها في الموجه (prompt) كقسم من نوع system، ثم — بعد اكتمال الرد — يكتب الدور عائدًا كحلقة (episode) لتتمكن المحادثات المستقبلية من استرجاعه.

إنه عميل JSON-RPC عبر WebSocket إلى Philia (`Sync.ConnectHandshake`, `Sync.MemoryQueryRequest`, `Sync.MemoryStoreRequest`, `Sync.MemoryDeleteRequest`). تُنشأ الاتصالات بشكل كسول (lazily)، وتُغلق عند أي خطأ وتُعاد عند الاستدعاء التالي؛ ويتدهور كل فشل بصمت و**لا يحجب مسار الدردشة أبدًا**.

## الإعدادات

يُتحكم في الـ gateway عبر ثلاثة متغيرات بيئة:

| المتغير | المعنى |
| --- | --- |
| `ARONA_MEMORY_URL` | عنوان WebSocket لخدمة scepter / Philia، مثلًا `ws://192.0.2.10:8424/ws`. |
| `ARONA_MEMORY_TOKEN` | token لخدمة الذاكرة. |
| `ARONA_MEMORY_WRITEBACK` | هل تُكتب الأدوار المكتملة عائدًا؟ الافتراضي **مفعّل**؛ اضبط `false` لتعطيله (يُحلل كقيمة منطقية صارمة — `0` غير مقبول). |

يجب تعيين كل من `ARONA_MEMORY_URL` **و** `ARONA_MEMORY_TOKEN` بقيم غير فارغة، وإلا يكون الـ gateway **معطَّلًا**: يُتخطى الاسترجاع والكتابة العائدة تمامًا ويُبلِّغ كل طلب عن `disabled`. يُرسل الـ token كمعامل استعلام `?token=` عند ترقية WebSocket وداخل طلب `Sync.ConnectHandshake` معًا.

```env
ARONA_MEMORY_URL=ws://192.0.2.10:8424/ws
ARONA_MEMORY_TOKEN=CHANGE_ME
ARONA_MEMORY_WRITEBACK=false
```

انظر [الإعدادات](configuration.md) للمرجع الكامل لمتغيرات البيئة.

## حقن الاسترجاع

مع تفعيل الـ gateway، يستعلم **كل دور دردشة** — REST غير المتدفق `/v1/chat/completions`، وREST المتدفق (SSE)، وRPC `chat.send` — خدمة الذاكرة قبل تمرير الطلب:

- الاستعلام هو **آخر رسالة مستخدم** في السياق المجمَّع.
- تُطلب حتى **5** ذكريات (`limit = 5`).
- تُعرض النتائج كقسم system بصيغة markdown بعنوان `## Relevant Long-Term Memories`، نقطة `- [score] text` لكل ذكرى (الدرجات بمنزلتين عشريتين، وتُتخطى الإدخالات الفارغة)، وتُسبَق إلى قائمة الرسائل كرسالة من نوع `system`. الحقن قادر على التكرار بأمان (idempotent): فالسياق الذي يحمل القسم بالفعل لا يُحقن مرة أخرى.
- إذا لم تُرجع أي ذكريات ذات صلة، فلا يُحقن شيء ويستمر الدور دون تغيير.

يجري الاسترجاع قبل حفظ المحادثة وتمريرها إلى الـ upstream؛ ولا تضيف خدمة الذاكرة البطيئة أو الفاشلة **أي ضمان كمون (latency)** يتجاوز مهلة RPC الخاصة بها البالغة 10 ثوانٍ، ولا يمكنها أن تفشل الطلب.

## الكتابة العائدة

بعد اكتمال رد المساعد، يُكتب الدور عائدًا إلى خدمة الذاكرة كعقدة **حلقة (episode)**. نص الحلقة هو نسخ استدلالي (heuristic transcript) للدور — `User: <user content>\nAssistant: <assistant content>` (يُحذف أي جانب فارغ؛ وإذا كانا فارغين تُتخطى الكتابة العائدة). الكتابة العائدة **تُطلق وتُنسى (fire-and-forget)**: تعمل في مهمة منبثقة (spawned task)، ولا تحجب استجابة الدردشة أبدًا، وتُسجَّل إخفاقاتها فقط داخل عميل الذاكرة. (في مسار REST المتدفق، تتطلب الكتابة العائدة بالإضافة إلى ذلك وجود محادثة مرتبطة بالطلب؛ بينما يكتب مسارا REST غير المتدفق وRPC عائدًا في كل الأحوال.)

## التحكم لكل طلب

يقبل كل من جسم طلب دردشة REST ومعاملات RPC `chat.send` حقلًا اختياريًا `memory` لتجاوز إعدادات الخادم **في كل استدعاء**:

```json
{ "model": "…", "messages": […], "memory": false }
```

- `memory: true` / `memory: false` — يفرضان تشغيل/إيقاف الاسترجاع لهذا الدور.
- محذوف (`null`) — يتبع إعدادات الخادم (`req.memory.unwrap_or(true)`)، أي مفعّل إذا وفقط إذا كان الـ gateway مكوَّنًا.

يؤثر التجاوز على الاسترجاع فقط؛ بينما تتبع الكتابة العائدة `ARONA_MEMORY_WRITEBACK` بالإضافة إلى كون الـ gateway مفعَّلًا.

## حالات الترويسة

تحمل استجابات REST حالة الذاكرة الخاصة بالدور في ترويسة الاستجابة **`X-Arona-Memory`**؛ وتعكس استجابة RPC `chat.send` القيمة نفسها في حقل `memory` ضمن نتيجتها. الحالات الممكنة:

| القيمة | المعنى |
| --- | --- |
| `enabled` | طُلبت الذاكرة، والـ gateway مكوَّن، ونجح الاسترجاع، وحُقنت ذكرى واحدة على الأقل. |
| `disabled` | الـ gateway غير مكوَّن، أو `memory: false` في الطلب، أو لا توجد رسالة مستخدم للاستعلام، أو نجح الاسترجاع لكنه لم يُرجع أي ذكريات ذات صلة (لا شيء يُحقن). |
| `offline` | طُلبت الذاكرة والـ gateway مكوَّن، لكن استدعاء الاسترجاع فشل (رُفض الاتصال، أو خطأ RPC، أو مهلة انتهت). |

```http
HTTP/1.1 200 OK
X-Arona-Memory: enabled
```

## دلالات الفشل

يتدهور كل شيء بشكل صريح وفي الاتجاه نفسه — الدردشة تعمل دائمًا:

- **فشل الاسترجاع** — يُسجَّل بمستوى `warn`؛ ويستمر الطلب دون ذكريات محقونة ويُبلِّغ عن `offline` في الترويسة.
- **فشل الكتابة العائدة** — يُسجَّل داخل عميل الذاكرة؛ ولا تتأثر استجابة الدردشة.
- **خدمة الذاكرة غير مكوَّنة** — يكون الاسترجاع والكتابة العائدة بلا تأثير (no-op)؛ ويُبلِّغ كل طلب عن `disabled`.

لا يوجد أي وضع يجعل انقطاع الذاكرة يفشل طلب دردشة أو يؤخره بما يتجاوز المهل المحدودة الخاصة بالعميل.

## سطح RPC

يُكشف عن أسلوبَي إدارة على سطح JSON-RPC (يتطلب كلاهما JWT؛ انظر [واجهة JSON-RPC API](../api/jsonrpc.md)):

**`memory.status`** — لقطة لحالة الـ gateway:

```json
{
  "enabled": true,
  "writeback": true,
  "events": [
    { "kind": "recall", "detail": "…query…", "at": "2026-01-01T00:00:00.000Z" },
    { "kind": "writeback", "detail": "…node id…", "at": "…" },
    { "kind": "error", "detail": "…", "at": "…" }
  ]
}
```

`events` مخزن حلقي (ring buffer) في الذاكرة للنشاط الأخير — أحداث الاسترجاع والكتابة العائدة والحذف والأخطاء، الأحدث أولًا، حتى العدد المطلوب (يطلب معالج الحالة آخر 50؛ بينما يبلغ الحد الأقصى للمخزن نفسه 100). وهو **ليس** سجل تدقيق دائم — يُصفَّر عند إعادة التشغيل.

**`memory.delete`** — يقتطع عقدة مخزَّنة حسب المعرّف:

```json
{ "node_id": "…" }
```

يُرجع `{ "deleted": true | false }`. ويفشل بخطأ عندما يكون `node_id` مفقودًا أو عندما لا تكون خدمة الذاكرة مكوَّنة.

## ذات صلة

- [الإعدادات](configuration.md) — متغيرات `ARONA_MEMORY_*`.
- [البدء السريع](quickstart.md) — إعداد شامل من البداية إلى النهاية للـ gateway.
- [الـ backends](backends.md) — كيف تُوجَّه طلبات الدردشة قبل تنفيذ الاسترجاع.
- [الفوترة والاستخدام](billing-usage.md) — كيف تُقاس أدوار الدردشة نفسها.
- [العمليات](operations.md) — السجلات والصحة لاتصال الذاكرة.
- [واجهة JSON-RPC API](../api/jsonrpc.md) — `memory.status`، `memory.delete`، `chat.send`.
- [نظرة عامة](README.md)

<!-- src: packages/core/src/memory/mod.rs:1-9 (design), 25-28 (section header), 32-48 (MemoryState), 82-98 (config), 212-241 (recall_and_inject), 342-376 (delete & writeback_dialogue); packages/core/src/gateway/server.rs:558-564 (X-Arona-Memory header), 1036-1040 (REST recall), 1085-1093 (REST writeback), 1269-1273 (SSE recall), 1401-1407 (SSE writeback); packages/core/src/gateway/rpc.rs:783-802 (memory.status / memory.delete), 1517-1519 (memory override), 1665-1672 (RPC recall), 1848-1858 (RPC writeback) -->

<!-- note: The fact sheet described the `disabled` header state as "gateway not configured, or `memory: false` on the request". Verified against source (memory/mod.rs:212-241): `disabled` is also returned when recall succeeded but produced no relevant memories to inject, or when the context has no user message to query. This page documents the full state machine. -->
