نعم، هذا هو **النص الأساسي النهائي** بعد الدمج والتنظيف والتوحيد، بصياغة يمكن اعتمادها كنسخة مرجعية مقفلة.

***

# FOX MQL5 | Decision Index

**الحالة:** قيد الإغلاق المتسلسل  
**المرجع:** FOX MQL5 | المرجع المعماري والبرمجي النهائي v1.0  
**القاعدة:** لا قيمة افتراضية، لا افتراض، ولا تنفيذ قبل قبول DEC-16-01 إلى DEC-16-12.

## حالات القرار
- `PROPOSED`: توصية جاهزة للمراجعة.
- `ACCEPTED`: اعتمدها المالك وأصبحت ملزمة للكود والاختبارات.
- `SUPERSEDED`: استبدلت بقرار أحدث مع حفظ الأثر التاريخي.
- `BLOCKED`: البيانات غير كافية ولا يجوز التخمين.

***

# DEC-16-01 | عقد Swing H1 واختيار الـAnchor

**الحالة:** ACCEPTED  
**تاريخ الاعتماد:** 2026-07-10  
**النطاق:** StructureReader، Fibonacci anchors، nearest structural levels، Opportunity classification  
**المالك:** Chief Implementation Architect، والاعتماد النهائي لمالك FOX

## A. Contract، السلوك المعماري الثابت
1. Swing H1 لا يكون صالحاً إلا إذا كان `Confirmed` من شموع H1 مغلقة فقط. لا تدخل الشمعة الحالية في التأكيد.
2. الكاشف يعيد Swing typed تحمل: High/Low، السعر، bar identity، confirmation time، algorithm version وquality facts.
3. لا يسمح بنقطتين من النوع نفسه داخل زوج Anchor.
4. زوج Anchor الرسمي نقطتان متعاقبتان زمنياً من نوعين متعاكسين، مؤكدتان، غير مبطلتين، وتحققان Eligibility Policy للإصدار النشط.
5. الاختيار حتمي ولا يعتمد على ترتيب traversal أو اجتهاد التنفيذ.
6. Fibonacci تستخدم زوج Anchor الرسمي نفسه ولا تعيد اختيار Swing خاصاً بها.
7. عند غياب زوج صالح أو نقص بيانات Eligibility تكون Structure غير صالحة والنتيجة `Wait`.
8. تحفظ Snapshot هويات الطرفين وسبب الاختيار ونسختي Contract وPolicy لضمان Replay.
9. `Eligibility Policy` بنيوية خالصة: تعتمد فقط على حقائق Market Intelligence مثل صحة الـSwing، اكتمال البيانات، جودة النقطة، الحد الأدنى للموجة، التسلسل الزمني وصلاحية الأسعار. يحظر عليها قراءة Opportunity أو Regime decision أو Engine أو Entry Method أو Risk/Execution state. لذلك ينتج StructureReader حقيقة Canonical واحدة مشتركة بين جميع المحركات.

## B. حسم تعدد الـSwings والـAnchor الرسمي
1. ترتب الـSwings المؤكدة تصاعدياً حسب `barTime`، ومع التعادل حسب `barIndex` ثم ترتيب نوع ثابت.
2. تضغط النقاط المتتالية من النوع نفسه: High ثم High يحتفظ بالأعلى، وLow ثم Low يحتفظ بالأدنى.
3. عند تساوي السعر ضمن `PriceEqualityTolerance`: يحتفظ بالأقدم كهوية بنيوية، ويسجل الأحدث كـliquidity retest.
4. تنشأ الأزواج المرشحة من كل نقطتين متجاورتين بعد الضغط، بشرط اختلاف النوع.
5. يسقط كل زوج لا يحقق Eligibility Policy أو أصبح invalidated.
6. يختار الأحدث اكتمالاً: الزوج ذو أحدث `endSwing.confirmationTime`.
7. عند التعادل: أحدث `endSwing.barTime`، ثم range الأكبر، ثم أقدم `startSwing.barTime`.
8. يبقى الزوج Anchor حتى يبطل أو يظهر زوج أحدث مؤهل؛ لا يستبدل بزوج أقدم لمجرد أن range أكبر.

**مبرر الاختيار:** `most-recent-eligible` تمثل آخر موجة مكتملة، وتحافظ على قرب مستويات H1 من السعر، وتمنع انتقاء موجة تاريخية بعيدة.

## C. Policy Values المرتبطة بإصدار الاستراتيجية

| Policy Key | القيمة المقترحة لـ`FOX-STRAT-1.0.0` | التصنيف |
| --- | --- | --- |
| `SwingPivotDepth` | 3 شموع لكل جانب | Strategy Policy |
| `SwingVolatilityPeriod` | ATR(14,H1) | Strategy Policy |
| `SwingMinRangeATR` | 1.0 | Strategy Policy |
| `SwingLookbackBars` | 300 H1 closed bars | Strategy Policy |
| `SwingMinBrokerDistanceFactor` | 3 × broker stop distance | Execution-aware Policy |
| `PriceEqualityTolerance` | 1 trade tick | Symbol Policy |

لا تثبت هذه القيم داخل المرجع المعماري ولا hard-code في Reader. تحملها `StrategyPolicyVersion` وتدخل في Audit وSnapshot وReplay. تغيير قيمة ينشئ إصدار Policy جديداً ولا يغير Contract.

## D. سبب اختيار الـContract
السلوك حتمي وقابل لإعادة التشغيل ولا يعيد الرسم بعد التأكيد. فصل الأرقام عن العقد يسمح بمعايرة الاستراتيجية وإصدارها دون كسر البنية أو تغيير معنى Reader.

## E. بدائل اختيار الـAnchor
### الأحدث المؤهل، التوصية
- **الميزة:** أقرب للسعر ومتوافق مع H1/M15 وحتمي.
- **العيب:** قد يستبدل موجة رئيسية بموجة أحدث أصغر إذا حققت Eligibility.
- **الأثر:** مستويات تشغيلية أحدث، وتعتمد الجودة على Eligibility Policy.

### أكبر Range داخل Lookback
- **الميزة:** موجة مهيمنة وثابتة.
- **العيب:** قد تبقى بعيدة عن السعر.
- **الأثر:** Fibonacci أطول عمراً ودقة أقل للمنطقة القريبة.

### أعلى Quality Score
- **الميزة:** يدمج الحداثة والحجم والسيولة.
- **العيب:** يحتاج أوزاناً ومعايرة وقد يخفي سبب الاختيار.
- **الأثر:** مرونة أعلى وتعقيد أكبر في Audit وReplay.

### Anchor حسب Regime
- **الميزة:** تخصيص الموجة للحالة.
- **العيب:** يجعل كل محرك يرى بنية مختلفة.
- **الأثر:** يكسر canonical structure fact، لذلك مرفوض في v1.

## F. الأثر على النظام
يثبت عقد StructureReader وFibonacciReader وsnapshot health، ويؤثر مباشرة في Trend/Breakout/Reversal وDashboard وReplay. لا يحدد وحده support/resistance universe، فهذا قرار DEC-16-02.

## G. التوصية النهائية
اعتماد Contract القسم A وخوارزمية `most-recent-eligible` في القسم B. تعتمد الأرقام حصراً كـPolicy Values للإصدار `FOX-STRAT-1.0.0`، وتتغير بإصدار موثق دون تعديل العقد.

***

# DEC-16-02 | Universe الدعم والمقاومة وترتيب المناطق

**الحالة:** ACCEPTED  
**تاريخ الاعتماد:** 2026-07-10  
**يعتمد على:** DEC-16-01 ACCEPTED  
**النطاق:** Structure، Fibonacci، Dashboard، Decision، Entry Planning، Risk

## A. السؤال المطلوب حسمه
ما المصادر التي يسمح لها بإنتاج Support/Resistance canonical، وكيف تدمج المستويات المتقاربة إلى Zones، وكيف يختار النظام nearest support وnearest resistance دون خلط قوة المستوى بموقعه؟

## B. التوصية المقترحة
اعتماد Universe متعدد المصادر لكن مفصول دلالياً:

1. `Structural Swing Levels`: أسعار الـSwings المؤكدة الصالحة من H1.
2. `Active Anchor Extremes`: طرفا Anchor الرسمي من DEC-16-01.
3. `Fibonacci Levels`: مستويات مشتقة حصراً من Anchor الرسمي.
4. `SMC Zones`: Order Blocks / Fair Value Gaps فقط بعد تثبيت عقود تعريفها في Policy مستقلة.
5. `Session/Previous Period Levels`: Previous Day High/Low وPrevious Week High/Low كحقائق سياقية مستقلة.

لا تدخل الأرقام النفسية أو Moving Averages أو Pivot Points التقليدية في v1، لأنها تضخم Universe دون دليل أنها تضيف edge فوق المصادر البنيوية.

## C. Contract الثابت
كل Level أو Zone تحمل: `sourceType`, `sourceId`, `priceLow`, `priceHigh`, `timeframe`, `createdAt`, `confirmedAt`, `freshness`, `touchCount`, `brokenState`, `qualityFacts`, `policyVersion`.

- المستوى ذو سعر واحد يمثل Zone بعرض صفري قبل normalization.
- لا يسمى المستوى Support أو Resistance بصورة ثابتة؛ يصنف بالنسبة إلى Reference Price في Snapshot.
- أسفل السعر = Support candidate، أعلى السعر = Resistance candidate.
- إذا كان السعر داخل Zone، تصنف `AtZone` ولا يجوز وصفها دعماً أو مقاومة قريبة تعسفياً.
- إذا كان Reference Price داخل Zone، فالمسافة إلى تلك المنطقة تساوي صفراً. وعند الحاجة إلى Support أو Resistance تالٍ، يستمر البحث خارج حدود المنطقة الحالية فقط.
- `Nearest` تعني أقل مسافة executable إلى الحد الأقرب من Zone، ولا تعني الأقوى.
- `Best/Strongest` حقل منفصل ينتج من Ranking Policy، ولا يحل محل nearest.
- broken/invalid/stale sources لا تدخل Active Universe، لكنها تبقى في Audit.
- المصدر المشتق لا يضاعف القوة: Fibonacci وAnchor اللذان يشتركان في السعر يسجلان confluence، لا مستويين مستقلين كاملَي الوزن.

## D. Zone Clustering
1. ترتب المستويات سعرياً.
2. تدمج المستويات التي تقع مسافتها ضمن `ZoneMergeTolerance` من Policy Values.
3. حدود المنطقة هي أدنى وأعلى سعر للمكونات، مع حد أقصى للعرض `MaxZoneWidth`.
4. إذا تجاوز الدمج MaxZoneWidth، يبدأ Cluster جديد عند أكبر gap داخلي.
5. تحفظ المنطقة جميع source IDs ولا تفقد provenance.
6. مستوى مكسور لا يحذف المنطقة كلها؛ يعاد تقييم كل مكون ثم المنطقة.

## E. Ranking منفصل عن Nearest
Quality Score مقترح للعرض والمقارنة فقط، لا لتغيير الحقيقة البنيوية:
- source reliability.
- recency/freshness.
- number and quality of reactions.
- multi-source confluence.
- distance-normalized relevance.
- penalty للكسر أو كثرة الاختبارات.

الأوزان الرقمية Policy Values قابلة للإصدار. Decision Engine يقرأ facts والـscore، لكنه لا يعيد بناء Universe.

`Source reliability` خاصية لنوع المصدر ونسخة Policy فقط، ولا تستنتج من ربحية الصفقات أو نتائج التداول. قياس الأداء قد يبرر إصدار Policy جديداً عبر governance، لكنه لا يعدل Universe أو أوزانه ذاتياً أثناء التشغيل.

## F. Policy Values المقترحة لـFOX-STRAT-1.0.0
لا توجد بيانات تاريخية أو symbol capability matrix كافية لاعتماد أرقام نهائية لـ`ZoneMergeTolerance`, `MaxZoneWidth`, touch/reaction thresholds أو ranking weights. لذلك حالتها `BLOCKED FOR NUMERIC ACCEPTANCE`، ويجب اشتقاقها عبر datasets منفصلة للفوركس والذهب والمؤشرات، لا قيمة موحدة عمياء.

التوصية لشكل السياسة، لا قيمتها: التعبير عن tolerance والـwidth بوحدة `trade ticks` مع سقف ديناميكي مرتبط بـATR(H1)، وتوفير Policy Profile حسب asset class.

## G. البدائل
### Fibonacci فقط
- **الميزة:** بسيط وحتمي.
- **العيب:** يختزل السوق في موجة واحدة.
- **الأثر:** Dashboard نظيفة لكن Decision تفقد structure/confluence.

### Swings + Fibonacci فقط
- **الميزة:** أفضل توازن وبأقل تعقيد.
- **العيب:** لا يمثل session liquidity أو SMC.
- **الأثر:** مناسب MVP قوي، ويمكن توسيعه دون كسر العقد.

### Universe كامل من البداية، التوصية المعمارية
- **الميزة:** رؤية مؤسسية وprovenance وconfluence.
- **العيب:** يحتاج عقود SMC ومعايرة ranking.
- **الأثر:** بنية نهائية أفضل، لكن تفعيل المصادر يكون تدريجياً عبر Policy flags.

### اختيار أقوى Level فقط
- **الميزة:** قرار بسيط.
- **العيب:** يخلط location بالقوة وقد يتجاهل خطراً قريباً.
- **الأثر:** غير مناسب للمخاطر والتنفيذ، ولذلك مرفوض.

## H. التوصية النهائية
اعتماد Contract متعدد المصادر مع فصل `Nearest` عن `Strongest`، وZone clustering يحفظ provenance. في أول إصدار تنفيذي يفعّل `Swings + Active Anchor + Fibonacci + Previous Day/Week`، وتبقى SMC sources معطلة حتى اعتماد عقودها. لا تعتمد الأرقام قبل توفير datasets حسب فئة الأصل.

***

# DEC-16-03 | Regime Taxonomy, Hysteresis and Persistence

**الحالة:** ACCEPTED  
**تاريخ الاعتماد:** 2026-07-10  
**يعتمد على:** DEC-16-01 وDEC-16-02 ACCEPTED  
**النطاق:** Market Intelligence، TradingContext، Opportunity Classifier، Decision Policy، Dashboard، Audit وReplay

## A. الفصل الدلالي الإلزامي
- `Regime` يصف كيف يتصرف السوق الآن، ولا يقرر لماذا ندخل.
- `Opportunity` تصف سبب صفقة محتملة: Trend / Breakout / Reversal / Wait.
- `Regime` حقيقة Canonical من Market Intelligence؛ أما Opportunity فهي مخرج Decision Domain.
- لا يجوز لـRegime Reader اختيار Engine أو Entry Method أو Direction أو Lot أو SL/TP.
- لا يجوز لأي Engine إعادة حساب Regime خاص به.
- Regime لا يفرض Opportunity ولا يتحول إليها عبر mapping مباشر. `TRENDING` لا يعني Trend Opportunity، و`COMPRESSING` لا يعني Breakout، و`EXPANDING` لا يعني Breakout أو Reversal، و`RANGING` لا يعني Reversal. العلاقة الإلزامية هي `Regime → Evidence` فقط، بينما Opportunity Classifier يجمعه مع بقية الأدلة المستقلة.

## B. Taxonomy المقترحة
يعيد Regime Reader حالة أساسية واحدة فقط لكل Snapshot:

1. `TRENDING`: حركة اتجاهية مستقرة ذات بنية وزخم واتساق كافٍ.
2. `RANGING`: توازن سعري داخل حدود بنيوية قابلة للتعريف، دون اتجاه مهيمن.
3. `COMPRESSING`: انكماش في المدى/التقلب والسيولة يرفع قابلية توسع لاحق، دون ادعاء اتجاه الاختراق.
4. `EXPANDING`: توسع نشط في المدى/التقلب بعد خروج من توازن أو ضغط، دون اعتباره Breakout Opportunity تلقائياً.
5. `DISLOCATED`: سوق غير منتظم أو انتقالي، مثل shock/gap/spread distortion أو حركة لا تسمح باستدلال مستقر.
6. `UNKNOWN`: الأدلة غير كافية أو متعارضة.

وتحمل الحالة بعداً اتجاهياً مستقلاً:

`BULLISH / BEARISH / NEUTRAL / UNDEFINED`.

**قيد:** الاتجاه لا ينشئ Regime جديداً؛ `TRENDING+BULLISH` تركيب typed، وليس enum منفصلاً. حالات Session وNews وLiquidity وVolatility تبقى Facts مستقلة ولا تضخم Taxonomy.

## C. Contract السلوكي الثابت
1. Regime يبنى فقط من Market Intelligence Facts المختومة، ولا يقرأ نتائج التداول أو الربحية أو أوضاع AUTO/MANUAL.
2. كل تقييم ينتج `RegimeEvidence` لكل مرشح، وحالة نهائية واحدة، وسبب اختيار/رفض typed.
3. `UNKNOWN` حالة صحيحة وليست خطأً يجب إخفاؤه. عند نقص الصحة أو coherence تكون النتيجة UNKNOWN وDecision تفشل مغلقة إلى Wait.
4. `DISLOCATED` تمنع الدخول الجديد افتراضياً عبر Decision Policy، لكنها لا توقف إدارة المراكز القائمة.
5. الانتقال لا يحدث بمجرد تفوق مرشح في Snapshot واحدة؛ يمر عبر Hysteresis وPersistence.
6. الحالة الحالية لا تستخدم كدليل سوقي لصحتها؛ يسمح بها فقط كذاكرة انتقال لمنع التقلب.
7. يحظر تعديل thresholds ذاتياً أثناء التشغيل. أي تغيير يحتاج Policy Version جديدة.
8. كل Snapshot تحفظ: الحالة المرشحة، الحالة المستقرة، الاتجاه، الأدلة، transition state، counters/timestamps وPolicy Version.
9. نفس Facts ونسخة Policy والحالة السابقة يجب أن تنتج النتيجة نفسها في Live وBacktest وReplay.
10. `Candidate Regime` مشتقة حصراً من Facts الموجودة في Snapshot الحالية. أما الذاكرة الزمنية فتدخل فقط إلى Transition Machine لاختيار `Stable Regime`، ولا تضيف score أو confidence أو evidence للمرشح.

## D. آلة الانتقال
`Candidate → Confirming → Stable → Weakening → Transitioning`.

- `Candidate`: مرشح جديد تجاوز Entry Threshold.
- `Confirming`: يستمر المرشح خلال مدة/عدد تقييمات Policy.
- `Stable`: أصبح Regime الرسمي.
- `Weakening`: الحالة الرسمية هبطت دون Hold Threshold، لكنها لم تفقد صلاحيتها بعد.
- `Transitioning`: انتهت صلاحية الحالة السابقة ولم يثبت بديل؛ Regime الرسمي يصبح UNKNOWN.

قواعد الانتقال:

1. لكل Regime حد دخول `EnterThreshold` وحد استمرار أدنى `HoldThreshold`.
2. لا ينتقل النظام مباشرة بين حالتين متنافستين؛ يجب أن يحقق البديل شروط الدخول والاستمرار.
3. إذا فقدت الحالة الحالية Hold ولم يثبت بديل، تكون النتيجة UNKNOWN، لا توريثاً متفائلاً للحالة القديمة.
4. `DISLOCATED` يسمح له بمسار دخول سريع مستقل عند تحقق Hard Anomaly Facts، لأن انتظار persistence طبيعي قد يكون خطراً.
5. العودة من DISLOCATED تتطلب Recovery Persistence كاملة وعودة data/spread/volatility إلى نطاق صالح.
6. عند تعادل مرشحين بعد Policy Resolution تكون النتيجة UNKNOWN.

## E. Evidence Families
تسمح Policy باستخدام عائلات أدلة مستقلة مع حفظ القيم الخام:
- Structure: HH/HL/LH/LL، slope، range boundaries، break/hold facts.
- Directional strength: اتساق الاتجاه والزخم، لا Indicator منفرد كحقيقة نهائية.
- Volatility state: normalized range/ATR expansion or contraction.
- Compression: تقلص النطاق والتقلب وتراكم الحدود.
- Expansion: range release وسرعة الحركة واستمراريتها.
- Liquidity/market quality: spread، gaps، tick activity وجودة الأسعار.
- Location: العلاقة مع Canonical Zones، دون تحويل Zone نفسها إلى Regime.

لا يجوز تكرار الدليل نفسه تحت أسماء متعددة لرفع الوزن. يسجل `evidenceFamilyId` لمنع double counting.

## F. Policy Values لـFOX-STRAT-1.0.0

| Policy Key | الحالة |
| --- | --- |
| أوزان Evidence لكل Regime | BLOCKED |
| Enter/Hold thresholds | BLOCKED |
| Confirmation persistence | BLOCKED |
| Weakening grace period | BLOCKED |
| DISLOCATED anomaly thresholds | BLOCKED |
| Recovery persistence | BLOCKED |
| اتجاه Bullish/Bearish thresholds | BLOCKED |
| تقييم cadence على M15/H1 | BLOCKED |

لا توجد Datasets مصنفة تكفي لاعتماد أرقام مؤسسية للفوركس والذهب والمؤشرات. يلزم Dataset وlabeling protocol منفصلان حسب asset class، مع walk-forward validation ومنع تسرب المستقبل.

## G. البدائل وآثارها
### Taxonomy مختصرة: Trend / Range / Unknown
- **الميزة:** بسيطة وسهلة الاختبار.
- **العيب:** تخلط Compression وExpansion مع الفرص وتفقد حالات الخطر.
- **الأثر:** MVP أسرع، لكن Decision Matrix تصبح محملة باستنتاجات مكررة.

### Taxonomy المقترحة ذات الست حالات
- **الميزة:** تفصل البنية عن الانتقال والخطر، وتدعم Trend وBreakout وReversal دون تحويل Regime إلى قرار.
- **العيب:** تحتاج بيانات مصنفة وسياسة انتقال دقيقة.
- **الأثر:** وضوح أعلى وAudit أفضل وتعقيد معايرة متوسط. هذه هي التوصية.

### Regime احتمالي متعدد الحالات
- **الميزة:** يحتفظ بعدم اليقين بدلاً من إجباره على enum واحد.
- **العيب:** يحتاج calibration ونماذج وعتبات تشغيل أكثر تعقيداً.
- **الأثر:** مناسب لاحقاً كEvidence Provider، لا كعقد v1 النهائي.

### Regime خاص بكل Engine
- **الميزة:** مرونة محلية.
- **العيب:** يكسر Canonical Context ويفتح تعارضاً بين المحركات.
- **الأثر:** مرفوض معمارياً.

## H. أثر القرار
- TradingContext تحمل Regime واحدة مستقرة مع أدلتها وحالة انتقالها.
- TradingContext تفصل صراحةً بين `candidateRegime` المشتقة من Snapshot الحالية و`stableRegime` الناتجة عن آلة الانتقال؛ ولا تخلط أدلة المرشح مع transition memory.
- Dashboard تعرض Stable Regime وCandidate وسبب UNKNOWN/DISLOCATED.
- Opportunity Classifier تستخدم Regime كدليل واحد ضمن القرار، لا كخريطة آلية إلى Engine.
- Audit وReplay يعيدان transition memory، لا Snapshot منفردة فقط.
- Shadow Mode يقيس transition churn ومدة الحالات قبل اعتماد الأرقام.

## I. التوصية النهائية
اعتماد Taxonomy ذات الست حالات والبعد الاتجاهي المستقل، مع Hysteresis وPersistence وUNKNOWN fail-closed وDISLOCATED safety path. تبقى جميع القيم الرقمية `BLOCKED` حتى بناء Datasets مصنفة حسب فئة الأصل واعتماد Policy Version مستقلة.

***

# DEC-16-04 | تحكيم الفرص والقرار النهائي

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-01 وDEC-16-02 وDEC-16-03 ACCEPTED  
**النطاق:** Decision Domain، Opportunity Classification، Arbitration، Dashboard، Audit، Replay

## A. العقد الأساسي
1. كل Snapshot صحيحة تنتج `CandidateEvaluationSet` ثابتاً يحتوي بالضبط على ثلاثة عناصر: `TREND` و`BREAKOUT` و`REVERSAL`.
2. كل Candidate يحمل `evidence` و`counterEvidence` و`eligibility` و`invalidation` و`ttl` و`expectedEdge` و`reasonCodes` و`contractVersion`.
3. يحظر على النظام اختصار التقييم بعد أول مرشح مؤهل، ويحظر حذف أي مرشح من السجل قبل الحسم.
4. النتيجة الرسمية تكون واحدة فقط: `OfficialOpportunityDecision`، وتكون إما نوعاً واحداً من الأنواع الثلاثة أو `WAIT`.
5. لا يجوز لأي محرك أو Builder إعادة تصنيف المرشحين أو التصويت عليهم؛ الحسم يتم فقط عبر `Decision Arbitration`.

## B. قواعد الحسم
1. تُصفّى المرشحات غير المؤهلة أولاً.
2. إذا بقي مرشح واحد فقط، فهو القرار الرسمي.
3. إذا بقي أكثر من مرشح، تُطبق سياسة مفاضلة حتمية على `expectedEdge` و`invalidationQuality` و`costImpact`.
4. إذا لم يتجاوز الفائز هامش القرار، فالنتيجة `WAIT`.
5. عند التعادل النهائي، تكون النتيجة `WAIT` ما لم تنص Policy معتمدة على قاعدة ترجيح صريحة ومقيدة.

## C. مبدأ التفسير
1. كل قرار رسمي يجب أن يبيّن: لماذا فاز المرشح المختار، ولماذا خسر كل بديل، وما مقدار الهامش الفاصل.
2. يجب أن يبقى `CandidateEvaluationSet` كاملاً في Audit حتى لو كانت النتيجة `WAIT`.

## D. الإضافات الإلزامية
### 1) عقد القيمة المتوقعة
يبقى عقداً فقط، وليس معادلة ثابتة.  
ينص العقد على أن القيمة المتوقعة تعتمد على:
- الحركة المتوقعة.
- تكلفة التنفيذ.
- مسافة الإبطال.
- إسقاط الهدف.

أما طريقة الدمج والأوزان والمعادلات الرياضية فتعتبر جزءاً من `Decision Policy` وليست جزءاً من العقد نفسه.

### 2) عقد جودة الإبطال
تعريف Typed لجودة الإبطال باعتبارها حقيقة مستقلة تعتمد على:
- الصلاحية البنيوية.
- الوضوح.
- كفاءة المسافة.

ولا يسمح بتحويلها إلى درجة عامة أو Score غير محدد.

### 3) الفصل بين الأهلية والترتيب
يجب النص صراحة على أن:
- `Eligibility` هي بوابة قبول أو رفض.
- `Ranking / Score` يستخدم فقط للمفاضلة بين المرشحين المؤهلين.

ولا يجوز أن يؤدي ارتفاع `Score` إلى اعتبار مرشح غير مؤهل صالحاً للتداول.

### 4) تعريف هوية القرار
إضافة تعريف رسمي لهوية القرار:

`DecisionIdentity = SnapshotId + DecisionPolicyVersion + ContractVersion + OfficialOpportunity`

وبذلك يصبح كل قرار فريداً وقابلاً للتتبع وإعادة التشغيل والتحليل.

***

# DEC-16-05 | عقد التحكيم النهائي

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-04 ACCEPTED  
**النطاق:** Decision Arbitration، Audit، Replay، Decision Index

## A. الغرض
هذا العقد يحدد كيف يُحسم التنافس بين المرشحين المؤهلين داخل `CandidateEvaluationSet`.  
هو ليس Resolver جديداً، بل عقد حاكم لآلية arbitration النهائية بعد اكتمال التقييمات.

## B. المدخلات والمخرجات
**المدخلات:**
- `CandidateEvaluationSet`
- `DecisionPolicyVersion`
- `TradingContext`

**المخرجات:**
- `OfficialOpportunityDecision`
- `winnerReason`
- `loserReasons`
- `margin`
- `tieFacts`
- `contractTrace`

## C. الإلزام
1. يجب أن تكون مجموعة المرشحين كاملة وثابتة.
2. لا يسمح بأي scoring خفي خارج العقد.
3. لا يسمح بأي short-circuit.
4. لا يسمح بأي randomness.
5. لا يسمح بأي arbitration يعتمد على engine behavior.

## D. السبب المعماري
الاسم `Decision Arbitration` يحافظ على أن القرار النهائي يتم بين مرشحين جاهزين، وليس من خلال إعادة بناء المنطق من جديد.

***

# DEC-16-06 | عقد تركيب المقترح

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-04 وDEC-16-05 ACCEPTED  
**النطاق:** Proposal Composition، Strategy Framing، Entry Preparation

## A. الغرض
هذا العقد يركب `TradeProposal` من القرار الرسمي، ولا يقرر الاتجاه ولا يعيد تصنيف السوق.  
الـProposal هو تمثيل تشغيلي للفرصة المختارة، يترجمها إلى نية تداول قابلة للتهيئة اللاحقة.

## B. المدخلات والمخرجات
**المدخلات:**
- `OfficialOpportunityDecision`
- `TradingContext`
- `MarketStructureSnapshot`
- `DecisionPolicyVersion`

**المخرجات:**
- `TradeProposal`
- `proposalType`
- `direction`
- `hypothesis`
- `invalidationReference`
- `targetReference`
- `proposalConfidence`
- `proposalVersion`

## C. الإلزام
1. لا يجوز للـ`Proposal Composer` أن يعيد تقييم المرشحين.
2. لا يجوز له أن يقرر Risk.
3. لا يجوز له أن يقرر Execution.
4. لا يجوز له أن يغير `OpportunityType`.
5. لا يجوز له أن يولد Entry مباشر إذا كانت البنية تتطلب مراحل لاحقة.
6. لا يقرأ السوق مباشرة.
7. لا يستدعي Readers.
8. لا يضيف أي Canonical Facts جديدة.
9. لا يعيد تقييم Opportunity.
10. لا يغير `TradingContext`.
11. لا يجوز له إنتاج `Canonical Facts` جديدة أو تعديل الحقائق المعتمدة.

## D. السبب المعماري
`Proposal Composer` هو الجسر بين القرار canonical وبين التخطيط التنفيذي، وهو أوضح من مصطلح `Strategy Engine`.  
بهذا لا يعود النظام engine-driven، بل decision-driven مع طبقة تركيب صريحة بعد الحسم.

***

# DEC-16-07 | عقد خطة الدخول

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-06 ACCEPTED  
**النطاق:** Entry Planning، Order Geometry، Execution Preparation

## A. الغرض
هذا العقد يحول `TradeProposal` إلى `EntryPlan` قابل للتنفيذ من حيث البنية، لكن ليس من حيث الإرسال للسوق بعد.  
هو يحدد أين وكيف ستُبنى نقطة الدخول، من دون تغيير جوهر الصفقة أو الإشارة الأصلية.

## B. المدخلات والمخرجات
**المدخلات:**
- `TradeProposal`
- `TradingContext`
- `ExecutionPolicyVersion`

**المخرجات:**
- `EntryPlan`
- `entryMode`
- `entryPrice`
- `stopReference`
- `initialTargetReference`
- `validityWindow`
- `executionConstraints`

## C. الإلزام
1. لا يجوز إعادة تفسير الفرصة أو `proposal`.
2. لا يجوز تعديل الاتجاه.
3. لا يجوز توليد position sizing.
4. لا يجوز إطلاق أوامر فعلية.
5. لا يجوز تجاوز حدود السلامة أو السياسة.

## D. invariant
إذا لم يمكن بناء `EntryPlan` صالح من `TradeProposal` و`TradingContext`، فالنتيجة `WAIT` أو `INVALID_ENTRY_PLAN` حسب سياسة الفشل.

***

# DEC-16-08 | عقد تقييم المخاطرة والتحقق منها

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-07 ACCEPTED  
**النطاق:** Risk Gate، Capital Protection، Pre-Execution Validation

## A. الغرض
هذا العقد يتحقق من أن `EntryPlan` يسمح بالمخاطرة ضمن الحدود المعتمدة قبل أي تنفيذ.  
هو Gate سابق للتنفيذ، وليس طبقة لاحقة عليه.

## B. المدخلات والمخرجات
**المدخلات:**
- `EntryPlan`
- `AccountState`
- `RiskPolicyVersion`
- `PortfolioConstraints`

**المخرجات:**
- `RiskAssessmentResult`
- `approved / rejected`
- `riskPerTrade`
- `maxExposure`
- `portfolioImpact`
- `riskDecisionReason`

## C. الإلزام
1. لا يجوز تعديل `EntryPlan`.
2. لا يجوز تعديل `Proposal`.
3. لا يجوز تعديل `Opportunity`.
4. لا يجوز تجاوز limits الفعالة.
5. إذا كانت البيانات ناقصة أو متعارضة، فالنتيجة `fail closed`.

## D. المسؤوليات
1. تقييم مستوى المخاطرة.
2. احتساب مؤشرات التعرض اللازمة للتحقق من توافق `EntryPlan` مع حدود المخاطرة المعتمدة.
3. إصدار قرار القبول أو الرفض.
4. عدم الادعاء بأنه مسؤول عن `Position Sizing` التنفيذي.

## E. invariant
لا يجوز بدء التنفيذ قبل نجاح `Risk Assessment & Validation`.

***

# DEC-16-09 | عقد التنفيذ

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-08 ACCEPTED  
**النطاق:** Broker Interaction، Order Submission، Fill Handling

## A. الغرض
هذا العقد ينفذ فقط ما تم اعتماده بعد `Risk Assessment & Validation`.  
هو لا يعيد بناء القرار، ولا يعيد تقييم المخاطرة، ولا يغير `EntryPlan`.

## B. المدخلات والمخرجات
**المدخلات:**
- `EntryPlan`
- `RiskAssessmentResult`
- `ExecutionPolicyVersion`

**المخرجات:**
- `ExecutionResult`
- `fillStatus`
- `slippage`
- `executionPrice`
- `rejectReason`

## C. الإلزام
1. لا يعيد بناء القرار.
2. لا يعدل `Proposal`.
3. لا يعدل `EntryPlan`.
4. لا يعدل قرار المخاطرة.
5. لا يعيد التحكيم بين الفرص.
6. وظيفته الوحيدة هي إرسال الأوامر والتعامل مع نتائج التنفيذ.

## D. السبب المعماري
التنفيذ مرحلة تشغيلية نهائية، وليس مرحلة قرار.

***

# DEC-16-10 | عقد المركز ودورة حياته

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-09 ACCEPTED  
**النطاق:** Position State، Management، Lifecycle

## A. الغرض
هذا العقد يعرّف `Position` ككيان مستقل عن `Trade` و`Proposal` و`EntryPlan`.  
`Lifecycle` يدير حالة `Position` فقط بعد إنشائها واعتمادها، وليس فكرة التداول المجردة.

## B. المدخلات والمخرجات
**المدخلات:**
- `ExecutionResult`
- `PositionManagementPolicy`
- `MarketState`
- `PositionState`

**المخرجات:**
- `Position`
- `PositionStateUpdate`
- `ManagementAction`
- `closeIntent`
- `modifyIntent`
- `holdIntent`

## C. الإلزام
1. لا يجوز لـ`Lifecycle` إدارة شيء غير `Position`.
2. لا يجوز فتح صفقات جديدة داخل `Lifecycle`.
3. لا يجوز إعادة تصنيف opportunity.
4. لا يجوز تعديل risk policy من الداخل.
5. لا توجد `Position` إلا نتيجة `ExecutionResult` ناجح.
6. `Position` كيان مستقل عن `Trade` و`Proposal` و`EntryPlan`.
7. يبدأ `Lifecycle` فقط بعد إنشاء `Position`.

## D. السبب المعماري
هذا يغلق الفجوة المتبقية في pipeline: أصبح لـ`Lifecycle` كيان محدد وواضح يديره.

***

# DEC-16-11 | عقد إجراءات السلامة

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-10 ACCEPTED  
**النطاق:** Safety، Fail Closed، Emergency Handling، Governance

## A. الغرض
هذا العقد يحدد `Safety Actions` بدل الاكتفاء بمفهوم `HALT` العام.  
الاستجابة تختلف مع شدة الحالة، مع بقاء المبدأ العام `fail closed`.

## B. إجراءات السلامة
- `WAIT`
- `FreezeNewEntries`
- `CancelPendingOrders`
- `BlockExecution`
- `EmergencyHalt`

## C. قواعد التفعيل
يُختار الإجراء وفق:
- `severity`
- `contract violation type`
- `market data integrity`
- `execution anomaly`
- `governance mismatch`

## D. الإلزام
1. لا يوجد علاج واحد لكل الأخطاء.
2. لا يجوز للطبقات الأدنى تجاوز safety action.
3. أي violation يجب أن يكون typed ومفسراً وقابلاً للتدقيق.

## E. invariant
`HALT` ليس الإجراء الوحيد، بل أحد إجراءات السلامة الممكنة.

***

# DEC-16-12 | قابلية التفسير وحوكمة إعادة التشغيل

**الحالة:** ACCEPTED  
**يعتمد على:** DEC-16-11 ACCEPTED  
**النطاق:** Explainability، Governance، Dataset Dependency، Replay Fidelity

## A. متطلب قابلية التفسير
كل قرار جديد أو تعديل مستقبلي يجب أن يجيب صراحةً:
- لماذا هذا القرار؟
- ما الأدلة؟
- ما البدائل؟
- لماذا رُفضت البدائل؟
- ما الأثر؟
- كيف يمكن إعادة تشغيله أو مراجعته؟

هذا ليس Feature.  
هذا متطلب معماري.

## B. بوابة الاعتماد على البيانات
أي thresholds أو weights أو margins أو ranking rules تبقى:
- `versioned`.
- `dataset-backed`.
- `walk-forward validated`.
- `asset-class specific`.

## C. Replay Invariant
أي `Replay` يجب أن يعيد استخدام `PolicyVersion` الأصلية المحفوظة مع القرار، لا أحدث policy متاحة.  
إذا تعذر تحميل policy الأصلية أو كان hash غير مطابق، تكون النتيجة `REPLAY_VERSION_UNAVAILABLE` أو `fail closed` حسب طبقة التشغيل.  
لا يجوز إعادة تفسير قرار تاريخي بسياسة أحدث.

## D. قواعد الحوكمة
1. لا يوجد `self-calibration` داخل runtime.
2. لا توجد قيم افتراضية خفية.
3. لا تغيير لمعنى العقد عبر policy update.
4. أي تغيير ذي معنى يحتاج Decision Record جديدة أو `superseding` record واضحة.

***

# المبادئ العامة

## 1. Snapshot Cycle Invariant
جميع العقود من DEC-16-04 حتى DEC-16-12 تعمل على Snapshot واحدة تحمل `SnapshotId` واحداً.  
ولا يجوز خلط بيانات Snapshot مختلفة داخل دورة قرار واحدة.

## 2. Single Writer Principle
لكل Artifact معماري مالك واحد فقط.

مثال:
- `CandidateEvaluationSet` ← Decision Layer
- `OfficialOpportunity` ← Decision Arbitration
- `TradeProposal` ← Proposal Composer
- `EntryPlan` ← Entry Plan
- `RiskAssessmentResult` ← Risk Assessment & Validation
- `ExecutionResult` ← Execution
- `Position` ← Position Contract

ولا يسمح لأي طبقة أخرى بتعديل هذا الـArtifact بعد إنشائه.  
إذا احتاجت طبقة لاحقة تغييراً، فإنها تنشئ Artifact جديداً ولا تعدل الكيان الأصلي.

## 3. Immutable Pipeline
كل مرحلة:
- تقرأ مخرجات المرحلة السابقة فقط.
- لا تعيد تعديلها.
- لا تعيد تفسيرها.
- ولا تعيد بناء Facts أو Decisions تم اعتمادها.

***

# التسلسل المعماري النهائي

Readers  
↓  
TradingContext  
↓  
Opportunity Classification  
↓  
Decision Arbitration  
↓  
Proposal Composer  
↓  
Entry Plan  
↓  
Risk Assessment & Validation  
↓  
Execution  
↓  
Position  
↓  
Lifecycle  
↓  
Safety  
↓  
Governance  
↓  
Explainability

***

# عناصر أوصي بحذفها نهائياً

بعد اكتمال المراجعة لا أرى مكاناً صحيحاً معمارياً للعناصر التالية:
- `Strategy Engine`.
- `Trend Engine`.
- `Event Engine` كمحرك قرار، مع الإبقاء على `Event Facts` أو `Event Readers` ضمن طبقة Market Intelligence عند الحاجة.
- `Execution Intent`.
- `Trade Mission`.
- `AI Layer` كطبقة مستقلة داخل Decision Pipeline.
- أي `Runtime Self-Calibration`.
- أي `Runtime Learning` أو `Online Adaptation` يغير سلوك العقود أو القرار أثناء التشغيل.
- أي `Reader` يعمل بعد بناء `TradingContext`.
- أي `Contract` يقرأ السوق مباشرة بعد إغلاق `TradingContext`.
- أي `Artifact` يملك أكثر من مالك كتابة.
- أي `Replay` يعيد تفسير القرارات باستخدام أحدث Policy بدلاً من النسخة التاريخية.

***

# الحكم النهائي

بعد إدخال هذه التعديلات، أعتبر أن `Decision Domain` أصبح مغلقاً معمارياً، ولا أرى حاجة لإضافة `DEC-16-13` أو قرارات جديدة قبل بدء التنفيذ.  
المعمارية أصبحت قائمة على مبادئ واضحة وثابتة:
- حقائق سوقية موحدة.
- قرارات حتمية قابلة للتكرار.
- عقود ثابتة وغير قابلة للتعديل بعد إنشائها.
- تنفيذ يعمل بمبدأ `Fail Closed`.
- إعادة تشغيل دقيقة باستخدام نفس البيانات والسياسات التاريخية.
- فصل كامل للمسؤوليات بين القراءة، واتخاذ القرار، وبناء المقترح، والتخطيط، وإدارة المخاطرة، والتنفيذ، وإدارة المركز.

بهذه الصورة، تصبح الوثيقة جاهزة للدخول في `Architecture Freeze` والانتقال إلى مرحلة التنفيذ دون وجود فجوات معمارية جوهرية.