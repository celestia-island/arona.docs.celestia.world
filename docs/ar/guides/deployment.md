---
title: "النشر"
description: "تثبيت arona-server من أصل إصدار (release asset) أو سكربت أو المصدر؛ تشغيله على عتاد مباشر مع systemd، أو في Docker Compose، أو تحت إشراف malkuth؛ وترقيته بأمان."
---

# النشر

تُوزَّع Arona كـ**ملف ثنائي واحد مكتفٍ ذاتيًا بلغة Rust** مبني من crate الـ
`_cli`. ينشر workflow الإصدارات الملف باسم
`arona-server-<tag>-linux-x86_64`
(`.github/workflows/release.yml`)، والملف الثنائي نفسه يؤدي دورين:

- **خادم API** — `serve` (الـ gateway؛ يطبّق `migrate` ترحيلات الـ schema
  صراحةً)؛
- **أدوات النماذج** — `install` (كشف العتاد) و`status` و`deploy
  <model>` و`download <repo>` و`connect <panel-url>`.

يتطلب الخادم PostgreSQL. يُنشأ الـ schema وتُطبَّق ترحيلاته تلقائيًا عند
الإقلاع، لذا فإن النشر في معظمه هو "احصل على الملف الثنائي، وجّهه إلى قاعدة
بيانات، وشغّله". ابدأ بـ[البدء السريع](./quickstart.md) لجولة شاملة من
البداية إلى النهاية، ثم عُد إلى هنا لمعرفة تخطيط بيئة الإنتاج.

## المتطلبات

- **Linux x86_64** لأصل الإصدار المُجمَّع مسبقًا؛ أي منصة يدعمها Rust 1.91+
  يمكنها البناء من المصدر.
- **PostgreSQL** — مثال Docker Compose يستخدم `postgres:16-alpine`؛
  أي إصدار حديث يعمل. يرفض الخادم الإقلاع دون `DATABASE_URL`.
- **`curl` و`python3` و`ca-certificates`** إذا كنت تستخدم سكربت التثبيت
  (على Debian/RedHat يثبّتها السكربت بنفسه عبر apt/dnf).
- موقع قابل للكتابة للملف الثنائي (مثل `/usr/local/bin`) و، إذا كنت
  تشغّل تنزيلات النماذج، مساحة قرص لذاكرة النماذج المؤقتة تحت `ARONA_DATA_DIR`.

## التثبيت

### من أصل الإصدار

نزّل أصل الإصدار الخاص بالـ tag الذي تريده واجعله قابلًا للتنفيذ:

```bash
export VERSION=v0.1.25   # or any released tag
curl -fsSL "https://github.com/celestia-island/arona/releases/download/${VERSION}/arona-server-${VERSION}-linux-x86_64" \
    -o /usr/local/bin/arona-server
chmod +x /usr/local/bin/arona-server
```

### من سكربت التثبيت

يضم المستودع `scripts/install.sh`: يحلّ أحدث وسم إصدار (release tag) من
GitHub API (أو يحترم التجاوز `ARONA_VERSION`)، وينزّل الأصل المطابق،
ويُثبّته باسم `arona-server` في `~/.local/bin` افتراضيًا (يمكن تجاوز
الدليل بـ`ARONA_BIN_DIR`)، ويطبع الخطوات التالية:

```bash
curl -fsSL https://raw.githubusercontent.com/celestia-island/arona/master/scripts/install.sh | bash
# or pin a version and a directory:
ARONA_VERSION=v0.1.25 ARONA_BIN_DIR=/usr/local/bin ./scripts/install.sh
```

### من المصدر

```bash
cargo build --release -p _cli
install -m 0755 target/release/_cli /usr/local/bin/arona-server
```

`cargo build --release -p _cli` هو بالضبط ما يبنيه workflow الإصدارات
والـ Dockerfile.

## الإعداد

كل الإعدادات تتم عبر متغيرات البيئة؛ يوثّق [مرجع
الإعدادات](./configuration.md) كل متغير، و`.env.example` في جذر المستودع
نقطة بداية عملية. الحد الأدنى المطلوب:

| المتغير | مطلوب؟ | الغرض |
| --- | --- | --- |
| `DATABASE_URL` | نعم | سلسلة اتصال PostgreSQL؛ يخرج الخادم إذا لم تُضبط. |
| `JWT_SECRET` | نعم | سر توقيع الـ token؛ يرفض الخادم التشغيل بسر التطوير المدمج ما لم يكن `MOCK_MODE=1`. |
| `ARONA_ADMIN_TOKEN` | موصى به بشدة | token bearer مشترك لمسارات `/api/admin/*`؛ بدونها تعيد هذه المسارات 401 دائمًا. |

اختياري: `ARONA_HOST` (الافتراضي `0.0.0.0`) و`ARONA_PORT` (الافتراضي `8420`)
و`ARONA_REGISTRATION_OPEN` (قيم صادقة (truthy) `1`/`true`/`yes`/`on` —
يفتح التسجيل؛ أول مستخدم مسجّل يصبح مشرفًا) و`ARONA_DATA_DIR` (جذر ذاكرة
النماذج المؤقتة) و`ARONA_MEMORY_URL` / `ARONA_MEMORY_TOKEN` /
`ARONA_MEMORY_WRITEBACK` (بوابة الذاكرة) و`ARONA_API_RATE_LIMIT_RPM` (حد
المعدل لكل مفتاح) و`RUST_LOG` (مرشح التتبّع).

## ترحيلات قاعدة البيانات

يتصل `serve` بقاعدة البيانات ويطبّق كل ترحيلات الـ schema المعلّقة عند
الإقلاع (`init_database` → `Migrator::up`)، فلا توجد خطوة نشر منفصلة.
لتطبيق الترحيلات صراحةً — مثلًا للتحقق منها قبل أول إقلاع — نفّذ:

```bash
arona-server migrate
```

يحتاج مستخدم قاعدة البيانات امتياز **`CREATE`** على الـ schema المستهدف،
لأن ترحيل الإقلاع ينشئ الجداول. لا يوجد ترحيل بيانات أبعد من الـ schema:
قاعدة البيانات *هي* الحالة، لذا انسخها احتياطيًا قبل الترقيات (انظر
[Upgrade and backup](#upgrade-and-backup)).

## العتاد المباشر مع systemd

ملف unit مثال (`/etc/systemd/system/arona.service`). **كل القيم السرية
أدناه عناصر نائبة — استبدل `CHANGE_ME` قبل الاستخدام:**

```ini
[Unit]
Description=Arona API server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=arona
Environment=DATABASE_URL=postgres://arona:CHANGE_ME@127.0.0.1:5432/arona
Environment=JWT_SECRET=CHANGE_ME
Environment=ARONA_ADMIN_TOKEN=CHANGE_ME
Environment=ARONA_HOST=0.0.0.0
Environment=ARONA_PORT=8420
Environment=RUST_LOG=info
# Optional:
# Environment=ARONA_REGISTRATION_OPEN=1
# Environment=ARONA_MEMORY_URL=ws://127.0.0.1:8424/ws
# Environment=ARONA_MEMORY_TOKEN=CHANGE_ME
# Environment=ARONA_MEMORY_WRITEBACK=1
# Environment=ARONA_DATA_DIR=/var/lib/arona
ExecStart=/usr/local/bin/arona-server serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

ثم فعّله وشغّله:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arona
curl -fsS http://127.0.0.1:8420/readyz   # expect {"status":"ok",...}
```

## Docker Compose

يضم جذر المستودع `docker-compose.yml` بخدمتين:

- **`arona`** — يبني الصورة من `Dockerfile` (وسم `arona:local`)،
  وينشر `${ARONA_PORT:-8420}:8420`، ويتطلب `DATABASE_URL` و`JWT_SECRET`
  و`ARONA_ADMIN_TOKEN` — يفشل Compose سريعًا بخطأ من نمط
  `:?` إذا نُقص أيٌّ منها من `.env`. يعمل فحص الصحة (healthcheck) الخاص به
  عبر `curl -fsS http://127.0.0.1:8420/readyz`.
- **`postgres`** — `postgres:16-alpine` ببيانات اعتماد عناصر نائبة فقط
  (`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`؛ كلمة المرور
  الافتراضية `change-me` — تجاوزها) وحجم مُسمّى (named volume)
  `arona-pgdata`.

```bash
cp .env.example .env
# Edit .env, e.g.:
#   DATABASE_URL=postgres://arona:CHANGE_ME@postgres:5432/arona
#   JWT_SECRET=CHANGE_ME
#   ARONA_ADMIN_TOKEN=CHANGE_ME
docker compose up -d
```

خدمة postgres المرفقة قابلة للوصول على المضيف `postgres` داخل شبكة
Compose. يبني `Dockerfile` كلاً من `_cli` و`_agent` في باني
`rust:1.91-slim-bookworm`، ويُثبّت `ca-certificates` و`curl` و`python3` في
بيئة تشغيل `debian:bookworm-slim`، وينسخ الملفين الثنائيين إلى الداخل،
ويُعرّض المنفذين 8420 (الخادم) و5790 (API المحلي للـ `_agent` المرفق)،
ويشغّل `arona-server serve` كنقطة دخول (entrypoint).

## النشر المُشرَف عليه مع malkuth (اصطلاح workspace)

في هذا الـ workspace يعمل arona تحت إشراف **malkuth** كخدمة
`arona-malkuth.service`. ينطبق النمط على أي خدمة تُنشر هنا:

- يشرف malkuth على عملية `arona-server serve` — يطلقها ويفحصها
  ويعيد تشغيلها عند الفشل ويُفرّغ الاتصالات قبل الإيقاف.
- كل خدمة مُشرَف عليها لا تُكشف إلا عبر **منفذ معلومات (info port)** خاص
  بالخدمة و**منفذ وكيل (proxy port)** خاص بها؛ الخدمة نفسها ترتبط
  بـ`127.0.0.1` ولا يمكن الوصول إليها مباشرة من الشبكة أبدًا.
- يُشغَّل المشرف بـ`--watch <deployment-path>`: عندما يتغيّر الملف الثنائي
  في ذلك المسار — مثلًا عندما تنسخ الترقية ملفًا جديدًا فوقه — ينفّذ malkuth
  **إعادة تشغيل متدرجة (rolling restart)**، مثيلًا واحدًا في كل مرة،
  مع تفريغ الاتصالات أولًا.

عواقب تشغيلية:

- اربط `ARONA_HOST=127.0.0.1` عند التشغيل خلف وكيل المشرف؛
  الوكيل هو نقطة الدخول الوحيدة المواجهة للشبكة.
- الترقية تعني "انسخ الملف الثنائي الجديد إلى مسار النشر": المراقب
  (watcher) يطلق إعادة التشغيل المتدرجة. يمكنك أيضًا إعادة تشغيل الوحدة
  المُشرَف عليها صراحةً.
- وجّه فحص الصحة الخاص بالمشرف إلى `/readyz` (أو `/api/health`)؛ انظر
  [Health probes](#health-probes).

## فحوص الصحة

يُكشف الخادم عن مجموعتي فحوص صحة غير مُصادَق عليهما (كلتاهما مغطاة أيضًا
في [دليل العمليات](./operations.md)):

- `GET /healthz` و`GET /readyz` (مرادفان) و`GET /v1/health` تُعيد
  `200` مع `{"status":"ok","version":<version>,"build_hash":<hash>,"models":<n>,"providers":<n>}`.
- `GET /api/health` يُعيد شكل plana `HealthResponse`: `status` و`version`
  و`kind` و`uptime` (بالثواني) و`network` و`build_hash` و`engine_version`.

وجّه موازنات الحمل والمشرفين وفحوص صحة الحاويات إلى `/readyz`؛ واستخدم
`/api/health` عندما تحتاج تفاصيل زمن التشغيل (uptime) والشبكة.

## الترقية والنسخ الاحتياطي

- **احتفظ بالملف الثنائي السابق.** قبل استبدال `arona-server`، انسخه
  جانبًا — `cp /usr/local/bin/arona-server /usr/local/bin/arona-server.<version>.bak` —
  حتى تتمكن من التراجع عن الملف الثنائي فورًا إذا فشل الإصدار الجديد في
  الإقلاع.
- **انسخ PostgreSQL احتياطيًا.** تحتفظ قاعدة البيانات بكل الحالة —
  الـ backends وعقد الـ agent والمستخدمون والمفاتيح والمحادثات والاستخدام —
  والتغيير التلقائي الوحيد على الـ schema هو ترحيل الإقلاع. نفّذ `pg_dump`
  على قاعدة بيانات `arona` قبل كل ترقية.
- **يحتاج مستخدم قاعدة البيانات إلى صلاحيات `CREATE`**، لأن الترحيلات تعمل
  عند الإقلاع؛ لا يمكن لدور للقراءة فقط إقلاع الخادم.
- **أوقف التشغيل بأناقة.** يفرّغ الخادم الاتصالات الجارية عند
  SIGINT/SIGTERM، لذا فضّل `systemctl restart arona` أو إعادة تشغيل من
  المشرف على `kill -9`.

<!-- note: The release binary is a single self-contained Rust binary with no runtime assets; the docs avoid claiming "static" linking because the build uses the default cargo profile (no crt-static). -->
<!-- src: packages/core/src/gateway/run.rs:40-162 -->
