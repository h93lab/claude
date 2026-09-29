# H93 — Brief شامل لتطوير Storefront Lens إلى منصة "رادار الفرص"

> ملخص كامل لنقاش جلسة "منصة تجميع مشاكل التطبيقات" (2026-09-29)، مع نتائج 3 إيجنتس بحثيين (نقد المنتج / المعمارية التقنية / حلقة التنفيذ) ومراجعة كود Storefront Lens الحالي. الهدف: نقل السياق كاملاً لجلسة Storefront لمناقشة تحويل المشروع إلى منصة H93.

---

## 1. الفكرة الأصلية والتشخيص

**الفكرة:** منصة شخصية (مستخدم واحد) مدعومة بالذكاء الاصطناعي تجمع شكاوى/اقتراحات/عقبات الناس عن تطبيقات الموبايل، تصنّفها وتجمّعها وتقيّمها إلى فرص مرتبة، وتولّد PRD/SPEC يُنفَّذ مباشرة بـ Claude Code. الباك إند Supabase. تسجيل دخول باسورد/PIN.

**التشخيص المتفق عليه من الإيجنتس الثلاثة (بدون ما يشوفوا بعض):**
- القيمة مش في جمع الشكاوى؛ القيمة في **قفل الدايرة**: فكرة → اختبار → بناء → إيراد.
- كل المنافسين (GummySearch, BigIdeasDB, PainOnSocial, Ideabrowser, Exploding Topics) أدوات "قراءة" فقط. ولا واحد بيقفل الدايرة دي. ده الـ moat الوحيد المتاح لشخص واحد.
- GummySearch (أفضل أداة في المجال) قفلت لعملاء جدد في نوفمبر 2025 لعدم الاتفاق مع Reddit على شروط الـ API ([المصدر](https://gummysearch.com/final-chapter/)). الدرس: "الجمع المستمر من السوشيال" غير مضمون؛ صمّم لمصادر قابلة للاستبدال.
- شكوى ≠ استعداد للدفع. الأدوات الحالية تقيّم بـ (التكرار × الحدة) وده بالضبط الإشارة الضعيفة.

---

## 2. القرار الأساسي: Storefront Lens هو المنصة. لا نبني منصة جديدة.

بعد قراءة كود `h93lab/storefront-` (الـ schema، `packages/core/src/sync.ts`، `analysis.ts`، `ai.ts`، أدوات MCP):

| المطلوب في H93 | الحالة في Storefront Lens |
|---|---|
| سحب App Store (RSS + fallback على web API لـ apps.apple.com) | ✅ موجود مع تقرير sync وتشخيص |
| سحب Google Play (google-play-scraper) | ✅ |
| تخزين التقييمات + تتبع التغييرات + تاريخ الـ rating + snapshots | ✅ |
| تصنيف AI (sentiment / 11 topic / label / kind) | ✅ لكن غير كافٍ (انظر §4) |
| Jobs queue مع retries و progress (`FOR UPDATE SKIP LOCKED`) | ✅ |
| MCP server (15 أداة، bearer token) | ✅ ميزة كبيرة لا يملكها أي منافس |
| Boards | ✅ |
| Diagnostics + doctor CLI | ✅ |
| تجميع عبر التطبيقات (cross-app clustering) | ❌ التحليل لكل تطبيق منفرد |
| استخراج إشارات الاستعداد للدفع / الحل المؤقت / المنافس المذكور | ❌ |
| جدول الفرص + scoring + بوابة القابلية للتنفيذ | ❌ |
| مقبرة الأفكار المقتولة + جدول النتائج | ❌ |
| توليد SPEC.md لـ Claude Code | ❌ |
| Embeddings / pgvector | ❌ |
| مصدر Reddit / لصق يدوي | ❌ (مؤجل عمداً) |

**الخلاصة:** ~60–70% من المعمارية الموصى بها موجودة وشغالة. أهم توصية من التقييم ("ابدأ من المنافسين مش من الفايرهوز") هي بالضبط ما يفعله Storefront.

---

## 3. الفلسفة: 5 تغييرات جوهرية

1. **ابدأ من المنافسين (incumbents) لا من الفايرهوز.** 20–50 تطبيق مدفوع في مجال واحد → تقييماتهم + الثريدات التي تذكرهم. "يدفع لـ X ويكره فيه Y" هي الإشارة الوحيدة الصعب تزييفها والقابلة للتربح.
2. **قيّم الالتزام لا الإزعاج** (rubric في §5).
3. **بوابة القابلية للتنفيذ** (8 شروط) قبل أي SPEC (§6).
4. **جدول النتائج هو الشاشة الأساسية** + مقبرة الأفكار حتى لا يعيد الـ AI ترشيح ما قُتل.
5. **تحقق رخيص قبل البناء** (≤5 أيام، ≤$50) بقاعدة قرار واضحة (§7).

---

## 4. الفجوة في التحليل الحالي (`packages/core/src/analysis.ts`)

- الـ `label` نص حر، والتجميع يتم بـ `lower(label)` + طلب من الموديل دمج المتشابه — يعمل داخل تطبيق واحد، مستحيل عبر 30 تطبيق.
- لا يوجد استخراج لـ: يدفع لمن، لماذا ألغى، ما الحل المؤقت، أي منافس ذُكر. هذه الإشارات تحمل ~60% من وزن الـ rubric.
- الملخص يُكتب "لمصمم يبني تطبيقاً منافساً" — جيد، لكن لا يوجد إخراج قابل للتنفيذ.
- `ai.ts` عميل OpenAI-compatible فقط. Anthropic لديها compatibility layer على `/v1/chat/completions` فسيعمل، لكن نخسر Batch API (خصم 50%) و prompt caching الصريح. غير مستعجل، لكن يُضاف Anthropic SDK client لاحقاً.

---

## 5. Rubric تقييم الإشارة (per review)

| الإشارة | الوزن |
|---|---|
| يدفع حالياً لمنافس (اسم التطبيق + السعر/الخطة) | 25 |
| ألغى / بدّل / "أبحث عن بديل لـ X" | 20 |
| وصف حل مؤقت (Excel، تطبيقين معاً، تصدير يدوي) | 15 |
| "كنت سأدفع لو…" صريحة | 10 |
| تأكيد عبر مصدرين+ ومن 3 أشخاص مختلفين+ | 10 |
| الحداثة (تناقص، نصف عمر 90 يوم) | 10 |
| صاحب الشكوى واضح أنه من الشريحة المستهدفة | 5 |
| الشحنة العاطفية | 5 (متعمَّد أنها قليلة) |

**أصفار إجبارية (×0):** شكوى من السعر فقط بلا ميزة ناقصة؛ خلل بنية تحتية سنشاركه؛ يتطلب صلاحيات لن نحصل عليها (SMS، background location، بيانات بنكية بدون ميزانية Plaid).
**ترجيح المصدر:** تقييمات المتاجر (post-purchase) ×1.3 مقابل Reddit venting.

---

## 6. بوابة القابلية للتنفيذ (8 شروط، كلها يجب أن تمر)

| # | الشرط | PASS | FAIL |
|---|---|---|---|
| 1 | نطاق شخص واحد | ≤4 أسابيع، ≤6 شاشات، ≤3 جداول، بدون native modules | native code / ML training / real-time sync |
| 2 | صلاحيات المنصة | يعمل ضمن حدود iOS/Android foreground؛ بدون Meta/TikTok/X scraping | background location، SMS، بيانات تطبيقات أخرى، API مغلق |
| 3 | بدون network effects | قيمة للمستخدم #1 وحده | marketplace / social / chat |
| 4 | التربح | نموذج واحد واضح؛ المستخدم يدفع فعلاً لمنافس أو حل مؤقت | "إعلانات لاحقاً"، B2B sales، رفض صريح للدفع |
| 5 | دليل الطلب | منافس ≥1k تقييم و ≤3.8★، أو ≥15 شكوى مستقلة من مصدرين+ في 90 يوم | صفر منافسين أو مصدر واحد |
| 6 | التوزيع | نفس الـ subreddits/الثريدات تسمح بالعرض، أو كلمة مفتاحية في المتجر بطلب واضح ومنافسين ضعفاء | إعلانات مدفوعة أو enterprise فقط |
| 7 | البيانات/القانون | بدون PII حساس (طبي، مالي، قُصّر) | أي منها |
| 8 | ملاءمة المؤسس | يمكنك استخدامه يومياً | لا يمكنك dogfood |

الفشل في 2 أو 3 أو 7 **دائم**؛ الباقي يُعاد النظر بعد 6 أشهر.

---

## 7. التحقق قبل البناء (≤5 أيام، ≤$50)

المنصة تولّد تلقائياً، المؤسس يوافق وينشر يدوياً:
- Landing page + waitlist من الاقتباسات الحقيقية (Vercel + PostHog).
- Fake-door pricing: زر "ابدأ تجربة 7 أيام — $4.99/شهر" يكشف "قريباً، سجّل".
- رد **علني** في الثريد الأصلي ("عملت نموذجاً أولياً، تحب تجرب؟") — لا DMs، لا سحب إيميلات.
- Concierge test لـ 3 مستخدمين.

**قاعدة القرار:** كمّل إذا ≥20 waitlist أو ≥5 ضغطات سعر أو ≥3 ردود "ابعت" خلال 5 أيام. غير ذلك اقتل وسجّل السبب.

---

## 8. المصادر (الترتيب النهائي)

| الأولوية | المصدر | ملاحظات (متحقق منها 2026-09-29) |
|---|---|---|
| 1 | App Store RSS | موجود. حد 500 تقييم/تطبيق/دولة. تقرير واحد غير مؤكد أن الـ feed أصبح فارغاً (سبتمبر 2026) — Storefront لديه fallback على web API بالفعل |
| 2 | Google Play scraper | موجود. الحزمة الأصلية مهجورة؛ استخدم الـ fork المكتوب بـ TypeScript (`s-h-a-d-o-w/google-play-scraper`) |
| 3 | Reddit | مؤجل. مجاني غير تجاري، 100 req/min، موافقة يدوية (Responsible Builder Policy). **مخاطرة:** إطلاق تطبيق تجاري من الداتا قد يُعتبر استخداماً تجارياً (غير مؤكد) |
| لاحقاً | X | pay-per-use $0.005/post. 20k بوست = $100/شهر، ضجيج عالٍ |
| ❌ | Facebook / G2 / Capterra / Product Hunt | Groups API محذوف منذ 2024؛ الباقي مخالف للشروط أو ضعيف الإشارة |
| ✅ | لصق يدوي (paste-import) | يُضاف كمصدر من الدرجة الأولى — أي مصدر قد يُغلق |

**حل حد الـ 500:** أضف نفس التطبيق بـ 3–4 دول (us, gb, au, ca). الـ schema يدعم ذلك بالفعل `unique (store, store_id, country)`.

---

## 9. المعمارية التقنية

- **الأساس الحالي يبقى:** pnpm monorepo، Next.js 16 + shadcn، worker (croner)، MCP (Streamable HTTP)، Postgres على Supabase عبر transaction pooler، صور WebP على القرص، Docker + Cloudflare Tunnel.
- **pgvector** على Supabase Postgres: `create extension vector` + جدول embeddings + HNSW index.
- **Embeddings:** `voyage-4` (1024-d، 200M توكن مجاناً على السلسلة 4) — عملياً مجاني لسنوات. بديل: OpenAI text-embedding-3-small عبر OpenRouter.
- **الموديلات:** `claude-haiku-4-5` للتصنيف بالجملة (Batch API + prompt caching؛ الحد الأدنى للـ cache على Haiku 4.5 = 4096 توكن فاجعل الـ taxonomy ≥4.1k). `claude-opus-5-5` للتحليل الأسبوعي وكتابة الـ SPEC.
- **Clustering تراكمي (بدون HDBSCAN في MVP):** عند كل embedding: أقرب 3 centroids؛ تشابه ≥0.80 ينضم (centroid running mean)؛ 0.65–0.80 يسأل Haiku "نفس الشكوى؟"؛ <0.65 cluster جديد. دمج ليلي للـ centroids ≥0.85.
- **التكلفة المقدرة:** AI ≈ $25/شهر (20k تقييم مصنَّف)، Supabase Pro $25 (إن لزم)، X اختياري $25–50. إجمالي **$50–100/شهر**.
- **الأمان:** المنصة الحالية خلف Cloudflare Access + MCP_TOKEN + Basic Auth اختياري. لو انتقلنا لـ Supabase Auth: signups مقفولة + RLS + PIN كفتح سريع على الجهاز فقط فوق session صالحة + TOTP. المفاتيح في `supabase secrets` (Vault لا يصل للـ Edge Functions).

---

## 10. Schema المقترح (migration 0004+)

```sql
create extension if not exists vector;

-- إثراء التصنيف
alter table reviews
  add column if not exists pain_score smallint,           -- 0..5
  add column if not exists wtp_signal text check (wtp_signal in
    ('paying_competitor','churned','workaround','stated_wtp','none')),
  add column if not exists competitor_mentioned text,
  add column if not exists workaround text,
  add column if not exists evidence_span text,            -- الاقتباس الحرفي الداعم
  add column if not exists raw_analysis jsonb;            -- لإعادة المعالجة عند تغيير الـ schema

create table review_embeddings (
  app_id uuid not null, review_id text not null,
  v vector(1024) not null,
  primary key (app_id, review_id),
  foreign key (app_id, review_id) references reviews(app_id, review_id) on delete cascade
);
create index on review_embeddings using hnsw (v vector_cosine_ops);

create table clusters (
  id bigserial primary key, label text, centroid vector(1024), n int default 0,
  distinct_apps int default 0, distinct_authors int default 0,
  score numeric, status text default 'open', updated_at timestamptz default now()
);
create index on clusters using hnsw (centroid vector_cosine_ops);

create table cluster_members (
  cluster_id bigint references clusters(id) on delete cascade,
  app_id uuid not null, review_id text not null, sim real,
  primary key (cluster_id, app_id, review_id)
);

create table opportunities (
  id bigserial primary key, cluster_id bigint references clusters(id),
  title text, score numeric, gate_results jsonb,
  status text default 'surfaced' check (status in
    ('surfaced','validating','building','shipped','killed')),
  spec_md text, outcome jsonb, created_at timestamptz default now()
);

create table killed_ideas (
  id bigserial primary key, opportunity_id bigint, embedding vector(1024),
  gate_failed smallint, reason text, killed_at timestamptz default now(), revisit_after timestamptz
);
create index on killed_ideas using hnsw (embedding vector_cosine_ops);

create table sources (id smallserial primary key, kind text, config jsonb, enabled bool default true, last_cursor text);
create table items (   -- عناصر من مصادر غير المتاجر (reddit/paste)
  id bigserial primary key, source_id smallint references sources, external_id text, url text,
  author_hash text, body text not null,
  content_hash text generated always as (md5(lower(regexp_replace(body,'\s+',' ','g')))) stored,
  posted_at timestamptz, fetched_at timestamptz default now(), status text default 'new',
  unique (source_id, external_id), unique (content_hash)
);
```

---

## 11. أدوات MCP الجديدة المقترحة

`list_opportunities` (مرتبة بالـ score، فلترة بالحالة) · `get_opportunity_evidence` (الاقتباسات + المصادر + الروابط) · `get_cluster` · `run_gate` (يطبّق الـ 8 شروط ويحفظ النتيجة) · `generate_spec` (يولّد SPEC.md) · `record_outcome` (installs, trial→paid, kill reason) · `kill_opportunity` · `import_items` (لصق يدوي).

الفكرة: Claude Code يكتب الـ SPEC وهو يقرأ الاقتباسات الحقيقية مباشرة من H93 عبر MCP.

---

## 12. صيغة SPEC.md التي ينفذها Claude Code بشكل أفضل

**Stack افتراضي للتطبيقات:** Expo (React Native + TypeScript) + Supabase + RevenueCat + EAS Build. Flutter فقط إن كان التطبيق animation-heavy. PWA فقط إن لم يلزم المتجر.

`docs/SPEC.md` ≤600 سطر:
1. المشكلة بكلمات المستخدمين: 5–10 اقتباسات حرفية بالرابط والتاريخ والـ rating؛ كل ميزة تشير إلى `Q-id`. ميزة بلا اقتباس تُحذف.
2. المستخدم المستهدف + لحظة الاحتياج (فقرة واحدة).
3. **بالضبط 5 مميزات** لكل منها acceptance criterion بصيغة Given/When/Then + خطوة تحقق يدوية.
4. **Non-goals صريحة** (Claude لا يستنتج من الحذف).
5. جداول Supabase كـ SQL + RLS لكل جدول.
6. الشاشات + navigation graph + ASCII wireframe.
7. مكان الـ paywall والسعر (منسوخ من المنافس) ومدة التجربة.
8. Definition of Done (EAS build، TestFlight، 5 ACs، typecheck أخضر).
9. خطة مهام كل واحدة ≤2 ساعة، تُنفَّذ واحدة واحدة في plan mode مع مراجعة بينها.

**CLAUDE.md للتطبيق (<200 سطر):** اقرأ SPEC قبل أي تغيير؛ لا مميزات خارج §3 (اقترح في BACKLOG.md)؛ مهمة واحدة لكل جلسة؛ لا dependencies جديدة بدون إدراجها في SPEC؛ typecheck + tests قبل "done"؛ اذكر Q-ids في commit messages. Pre-commit hook للـ typecheck.

---

## 13. حلقة التغذية الراجعة

- كل تطبيق تطلقه يُسجَّل كـ `own_app`: تقييماته + إيميلات الدعم + TestFlight feedback تدخل نفس الـ pipeline موسومة `own_app` وتنتج **iteration cards** بدل opportunity cards.
- قبل أي scoring: فحص تشابه ضد `killed_ideas`؛ تشابه >0.85 يظهر "قُتلت سابقاً: السبب" ويُكبت.
- نتائج التحقق (signups, conversion) تعود كـ labels لإعادة معايرة أوزان الـ score كل ربع.

---

## 14. الإيقاع وإدارة المحفظة

- **كل ربع:** تحقق من 4 فرص، ابنِ 2، أطلق 2. لا بناءين بالتوازي.
- **اقتل بعد الإطلاق (30 و60 يوم):** <100 تنزيل من قناة الإطلاق، أو <2% بدء تجربة، أو <1 دافع لكل 100 تنزيل بيوم 60.
- **الفائز:** ≥3% trial→paid + تقييمات عضوية تذكر الألم الأصلي.
- **هدف واقعي:** $1–50K MRR للتطبيق. 82% من الـ micro-SaaS لا تصل $5K/شهر؛ دور H93 رفع نسبة الإصابة لا إيجاد unicorn.
- **حتى لا تصبح الأداة هي المشروع:** جمّد مميزات H93 لربع؛ تُضاف ميزة فقط عند ألم حقيقي أثناء بناء تطبيق. شغل H93 يوم الجمعة فقط.

---

## 15. أسباب الفشل الخمسة والحلول

| السبب | الحل |
|---|---|
| بناء الرادار بدل التطبيقات | سقف أسبوعين لكل مرحلة؛ ممنوع مصدر جديد قبل إطلاق أول تطبيق |
| انهيار مصدر (Reddit denial، RSS فارغ، scraper مكسور) | adapter interface + أرشيف raw JSON + لصق يدوي + تنبيه عند أيام بصفر صفوف |
| clusters عامة ("تطبيق ميزانية أفضل") | ارفض أي cluster بلا منافس محدد + شريحة محددة + ميزة ناقصة محددة |
| فلتر "غير قابل للبناء بواسطتك" مفقود | ملف قدرات المؤسس (stack، لا ML، لا hardware) يُطبَّق عند التقييم لا بعده |
| لا ground truth | 200 عنصر موسومة يدوياً؛ قياس دقة المصنّف شهرياً؛ `raw_analysis` jsonb لإعادة المعالجة |

---

## 16. خطة التنفيذ على repo الـ Storefront

**المرحلة 1 — إثراء التصنيف (أسبوع)**
- migration `0004`: الأعمدة الجديدة على `reviews` (§10).
- تعديل `SYSTEM` prompt في `analysis.ts` لإخراج JSON صارم بالحقول الجديدة؛ الـ label بصيغة ثابتة `<ميزة ناقصة> — <سياق>`.
- إعادة التحليل للموجود (`analysed_at = null`).
- اختبارات في `packages/core/test/ai.test.ts` للـ normaliser الجديد.

**المرحلة 2 — التجميع عبر التطبيقات (أسبوع)**
- `create extension vector`؛ جداول `review_embeddings`, `clusters`, `cluster_members`.
- embedding client (voyage-4 أو عبر OpenRouter) في `ai.ts`.
- job `cluster_reviews` في الـ worker (nearest-centroid تراكمي).
- صفحة `/opportunities` في الـ web.

**المرحلة 3 — الفرص والبوابة (أسبوع)**
- جداول `opportunities`, `killed_ideas`؛ حساب الـ score من الـ rubric؛ الـ gate الثمانية؛ فحص المقبرة.
- أدوات MCP الجديدة (§11).

**المرحلة 4 — الإخراج والحلقة (أيام)**
- زر "Generate SPEC" → `SPEC.md` بالصيغة (§12).
- تسجيل `own_app` + iteration cards + `record_outcome`.

**المرحلة 5 (لاحقاً)** — مصدر Reddit كـ corroboration فقط؛ لصق يدوي؛ Anthropic SDK client مباشر للـ Batch API.

---

## 17. أسئلة مفتوحة للنقاش في جلسة Storefront

1. الاسم: H93 (المنصة كلها) أم H93 كمظلة و Storefront Lens يبقى اسم الـ module؟
2. الـ embedding provider: voyage-4 مباشرة (client جديد) أم OpenRouter لتوحيد الـ config الحالي؟
3. هل نُبقي المصادقة الحالية (Cloudflare Access + token) أم ننتقل لـ Supabase Auth + PIN؟ التوصية: أبقِ الحالية — أبسط وأكثر أماناً لمستخدم واحد.
4. المجال الأول (vertical) للـ 20–50 تطبيق: يُفضَّل مجال أنت مستخدم فيه، بعيد عن الصحة/البنوك. ترشيح vendor غير مؤكد: meditation، diet tracking، parenting.
5. هل ننقل التحليل الحالي (per-app insights) ليصبح مشتقاً من الـ clusters بدل الـ labels الحرة؟ التوصية: نعم، بعد المرحلة 2.

---

## 18. نتيجة النقاش مع "مالك كود Storefront" (إيجنت قرأ الكود الفعلي) — قرارات معدَّلة

### ما تغيّر في الخطة
1. **ترتيب المراحل انقلب:** الـ embeddings/pgvector تأجلت للمرحلة 4 (فقط إذا ثبت أن تجميع الـ labels غير كافٍ). 30 تطبيق × أعلى labels ≈ 600 label منظَّم؛ دمج LLM ليلي واحد عبر المجال كافٍ لاختبار وجود فرص. الترتيب الجديد: (1) إثراء التصنيف + إعادة التحليل + dedupe متعدد الدول → (2) `opportunities`/`killed_ideas` + evidence view + أدوات MCP للقراءة → (3) حلقة النتائج (`own_app`, `record_outcome`) → (4) pgvector عند الحاجة → (5) مصادر جديدة.
2. **الـ rubric يُعاد تصميمه:** 45/100 نقطة تعتمد على اسم منافس/سعر/إلغاء، وتقييمات المتاجر نادراً تذكر منافسين → توقّع `wtp_signal='none'` في ~90%. نحتفظ بالحقول كاستخراج (هي الـ moat) لكن الترتيب يُحسب بـ SQL من حقائق: عدد الـ listings المميزة، recency half-life (في SQL لا بالموديل)، نسبة 1–2★؛ و WTP كمضاعِف. حُذف "مصدرين+" (مستحيل قبل المرحلة 5) و"شريحة مستهدفة" (غير قابلة للقياس).
3. **أدوات MCP لا تشغّل LLM داخل الـ server:** `run_gate`→`save_gate_result`، `generate_spec`→`save_spec`. Claude Code هو من يحكم ويكتب باستخدام `get_opportunity_evidence`. إضافة `list_clusters` و `search_reviews` عبر التطبيقات بفلتر `wtp_signal`. أي عمل LLM طويل في الـ web يُمرَّر عبر `enqueue()` ويعيد job id (نمط `actions.ts:62-66`).
4. **Embeddings عند الحاجة: لا Voyage client جديد.** Voyage `/v1/embeddings` بصيغة OpenAI؛ نضيف `embed(cfg, texts, {inputType})` بجوار `chat()` في `ai.ts` + `settings.ai.embedding = {baseUrl, apiKey, model, dimensions}`. نخزّن `model` على `review_embeddings` (تغيير الموديل يُبطل كل المتجهات)، `dimensions: 1024` صريحة (text-embedding-3-small افتراضياً 1536). نُضمِّن `label + evidence` الإنجليزي المنظَّم لا النص الخام متعدد اللغات.
5. **المصادقة: تبقى Cloudflare Access + MCP_TOKEN (+ Basic Auth).** التطبيق يتصل بدور `postgres` فـ RLS لا ينطبق أصلاً (0001 يفعّله فقط لحجب PostgREST). Supabase Auth يفرض client SDK على codebase لا تستورد `@lens/core` في الـ client. PIN خلف Access "مسرح".
6. **`insights` per-app تبقى** كمسار رخيص؛ بعد المرحلة 2 نستبدل فقط نداء `CLUSTER_SYSTEM` (`analysis.ts:70-73,154-161`) بـ `group by` deterministic.

### أخطاء في الـ brief الأصلي تم تصحيحها
- **حيلة الدول المتعددة صحيحة لـ iOS فقط.** تقييمات Google Play غير مقيّدة بالدولة (`android.ts:112` id عالمي) → نفس التقييمات ×4 وتضخيم `n`/`distinct_apps`. الصور تُخزَّن `${appId}/screens/<hash>` (`sync.ts:136`) → 4 نسخ WebP متطابقة. الحل: العدّ بـ `(store, store_id)` لا `app_id`، ومنع multi-country للأندرويد (المهم هناك `lang`).
- `cluster_members` بلا FK إلى `reviews` → orphans عند حذف تطبيق؛ يُضاف بـ cascade. عدّادات `clusters.n/distinct_apps/distinct_authors` عرضة للانجراف → view. `distinct_authors` بلا معنى عبر التطبيقات (nickname).
- `killed_ideas.embedding` لعنوان مقارنة بـ centroids تقييمات = توزيعان نصيان مختلفان → خزّن centroid الـ cluster المقتول.
- `items.unique(content_hash)` عالمياً يتصادم على نصوص قصيرة ("doesn't sync") → scope بالمصدر أو حذف.
- العتبات 0.80/0.65/0.85 خاصة بالموديل → settings، وتُعاير على 200 عنصر موسومة قبل أي clustering.
- pgvector مع postgres.js: لا OID لـ `vector` → مرّر `JSON.stringify(float[])` مع `::vector`؛ `create extension vector with schema extensions` على Supabase؛ `set local hnsw.ef_search` داخل `sql.begin` (transaction pooler يعطي backend مختلفاً لكل statement).
- Jobs: `claimJob` يُسلسِل فقط الـ jobs التي تشترك في `payload.appId` (`jobs.ts:102-103`)، و `enqueue` يمنع التكرار للـ queued فقط (`jobs.ts:77`) → job عالمي قد يعمل مرتين بالتوازي ويتسابق على الـ centroid. الحل: job لكل تطبيق (`embed_app` بعد `analyse_app`) أو `pg_advisory_xact_lock`. لا job أحادي: timeout 15 دقيقة (`index.ts:210`) + `requeueStale(30)` سيقتله ويعيده.
- Migrations تعمل عند بدء الـ worker في transaction واحدة → 0004+ DDL فقط، لا backfill.

### المرحلة 1 — خطة diff دقيقة
**مصيدتان:** (أ) إعادة التحليل لا تحدث وحدها: الـ worker يضع `analyse_app` فقط عند `newReviews > 0 || !getInsights` (`worker/index.ts:250`)، و `upsertReviews` يصفّر `analysed_at` فقط عند تغيّر النص (`sync.ts:339`) → عمود `analysis_version` + job `analyse_all`. (ب) حجم الإخراج: 40 تقييم × 5 حقول جديدة مع `evidence_span` حرفي يتجاوز `maxTokens: 6000` (`analysis.ts:64`) → batch 20 أو 12k.

**الملفات:** `db/migrations/0004_review_signals.sql`؛ `packages/core/src/analysis.ts` (SYSTEM, `Classified`, `normaliseItem`, unnest update, pending query, batch 20)؛ `jobs.ts` (`JobType += "analyse_all"`)؛ `apps/worker/src/index.ts` (case جديد؛ شرط السطر 250 → pending count من `queries.ts:211`)؛ `queries.ts` (+فلتر `signal`، أعمدة جديدة)؛ `apps/mcp/src/tools.ts` `get_reviews`؛ `apps/web/app/actions.ts` `reanalyseAllAction` + زر في Settings؛ `index.ts` exports.

```sql
-- 0004_review_signals.sql (DDL فقط)
alter table reviews
  add column if not exists analysis_version smallint not null default 0,
  add column if not exists pain_score smallint check (pain_score between 0 and 5),
  add column if not exists wtp_signal text check (wtp_signal in ('paying_competitor','churned','workaround','stated_wtp','none')),
  add column if not exists competitor_mentioned text,
  add column if not exists workaround text,
  add column if not exists evidence_span text,
  add column if not exists raw_analysis jsonb;
create index if not exists reviews_signal on reviews (wtp_signal) where wtp_signal <> 'none';
```
Pending query: `where analysed_at is null or analysis_version < ${ANALYSIS_VERSION}` (const = 2). `analyse_all` يضع `analyse_app` لكل تطبيق لديه صفوف معلّقة.

**SYSTEM prompt:** لكل عنصر `id, sentiment, topic, kind, label, wtp_signal, competitor, workaround, evidence, pain`. القيود: `label` = `"<missing capability> — <context>"` ≤60 حرفاً، noun phrase إنجليزية، بلا أسماء تطبيقات؛ `competitor` فقط إن سُمّي تطبيق صراحة وإلا null؛ `workaround` ≤80 أو null؛ `evidence` substring حرفي من المدخل ≤200 أو null؛ `pain` 0–5. الـ normaliser يفرض الـ enums والقطع، ويرفض `evidence` غير موجود عبر `includes()` في النص الأصلي (يجعله null ويحتفظ بالصف)، ويخزّن العنصر الخام في `raw_analysis`.

**الاختبارات:** `ai.test.ts` — fallbacks للـ enums، التحقق من substring الـ evidence، قطع الـ label، `competitor` → null إن لم يكن string. `integration.test.ts:293` — mock provider يعيد الحقول الجديدة → الأعمدة و `raw_analysis` مكتوبة، `analysis_version` مرفوع، `analyse_all` يضع job واحداً لكل تطبيق به صفوف معلّقة ولا شيء عند عدم وجودها.
