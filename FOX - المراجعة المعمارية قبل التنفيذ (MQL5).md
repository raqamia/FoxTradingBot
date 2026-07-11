# FOX - المراجعة المعمارية قبل التنفيذ (MQL5)

# FOX - المراجعة المعمارية قبل التنفيذ (MQL5)
## حالة الوثيقة
**النوع:** مراجعة معمارية قبل التنفيذ
**النطاق:** دراسة المستند المرفق `arc.md` فقط
**الحالة:** غير معتمد للتنفيذ بعد
**قاعدة العمل:** Fail Closed. لا كود، ولا افتراضات لسد الفراغات.
> **ملاحظة حاكمة:** المستند المرفق يجمع بين أفكار معمارية، مقتطفات محادثة، وادعاءات سابقة بأن تعديلات برمجية تمت. لم تُرفق الشفرة أو فهرس قرارات رسمي أو عقود أنواع قابلة للتحقق. لذلك أتعامل مع العبارات من نوع «تم التنفيذ» و«تم التحقق» على أنها **سجل محادثة غير مُثبت**، وليست حقيقة معمارية أو دليل مطابقة.
* * *
# 1\. فهم المعمارية
## 1.1 Market Reading / Market Intelligence
هذه الطبقة هي المالك الوحيد لاستخراج **حقائق السوق** من الأسعار والسلاسل الزمنية والمؤشرات. نطاقها المقترح في المستند يشمل:
*   Trend.
*   Regime.
*   Structure / Location.
*   Liquidity.
*   Momentum.
*   Volatility.
*   Session.
*   SMC Blocks.
*   Fibonacci.

الفصل الزمني المقصود واضح على مستوى الفكرة:
*   **H1** لبناء Structure واستخراج آخر swing معتبر وشبكة Fibonacci ومستويات الدعم والمقاومة البنيوية.
*   **M15** للمراقبة التشغيلية قرب مستويات H1، مثل الاقتراب والتأكيد والاختراق والارتداد.
*   **H4 ليس الفريم الافتراضي** في هذا المسار.

وظيفة هذه الطبقة هي إنتاج facts فقط، لا اختيار صفقة، لا تحديد حجم، ولا إرسال أوامر. أقرب دعم ومقاومة ومنطقة الاهتمام هي مخرجات قراءة، وليست توصية تداول.

**توضيح معماري:** عبارة «المسافات محسوبة على H1» تحتاج دقة لغوية. المستوى يُستخرج من H1، لكن المسافة نفسها فرق سعري بين مرجع سعر محدد وبين مستوى H1. الفريم لا يحسب المسافة بذاته. يجب لاحقاً تحديد مرجع السعر: Bid أو Ask أو Mid أو Last أو إغلاق M15 المكتمل.
## 1.2 TradingContext
TradingContext هو عقد النقل بين القراءة والقرار. يحتوي Snapshot واحدة مختومة تشمل:
*   canonical facts.
*   typed values.
*   snapshot health.
*   structure timeframe.
*   monitor timeframe.
*   هوية زمنية تسمح بمعرفة لحظة الالتقاط ومدى حداثة البيانات.

بعد الختم لا يحق لأي طبقة لاحقة تعديل الحقائق أو إعادة تفسير مصدرها. القرار والمخاطر والتنفيذ تقرأ Snapshot ولا تكتب فيها.

**حد الملكية:** Market Reading تبني Snapshot عبر Builder أو مرحلة تجميع، ثم جهة واحدة تختمها وتنشرها. بعد النشر تصبح Read Only منطقياً.
## 1.3 Decision Authority
هذه هي سلطة القرار الوحيدة. تقرأ Snapshot المختومة، ثم:

1. تتحقق من أهلية السياق للقرار.
2. تحدد نوع النتيجة: Trend أو Breakout أو Reversal أو No Trade.
3. تختار محركاً واحداً فقط وفق السياسة والنظام السوقي.
4. تستقبل proposal من المحرك المختار.
5. تصدر **TradeProposal واحداً أو NoTradeDecision واحداً** لكل دورة قرار.

التوزيع المقصود:
*   Trend Engine لصفقات الاتجاه.
*   OCO / Compression Engine للاختراق أو الارتداد.
*   Manual OCO طلب من المستخدم، لكنه لا يتجاوز القرار أو المخاطر أو التنفيذ.

**توضيح معماري:** المحركات ليست سلطات تنفيذ. هي مولدات مقترحات تحت سيطرة Decision Authority. Score وConfidence داخل المحرك لا يعطيانه حق الإرسال.

**قاعدة single-engine-by-regime:** لا تعمل المحركات الخمسة أو الاستراتيجيات المتعددة بالتوازي لتنتج أوامر متنافسة. Policy/Arbiter يختار مساراً واحداً أو يمنع التداول.
## 1.4 Risk / Sizing / Portfolio
هذه الطبقة تحول Proposal إلى Mission قابلة أو غير قابلة للتنفيذ. مسؤولياتها:
*   اشتقاق أو التحقق من SL وTP.
*   حساب حجم الصفقة وفق المخاطرة.
*   التحقق من الحد الأدنى للثقة وRR.
*   فحص التعرض الحالي والجديد.
*   فحص حدود اليوم والحساب والمحفظة.
*   منع أي requested lot يسبب upsizing فوق الحد المسموح.
*   رفض مغلق عند نقص أي مدخل مطلوب.

المخرج يجب أن يكون أحد اثنين:
*   Approved Mission ثابتة ومكتملة وقابلة للتدقيق.
*   Risk Rejection برمز سبب واضح.

لا يحق للتنفيذ إعادة حساب اللوت أو تعديل المخاطرة لتحويل الرفض إلى قبول.
## 1.5 Execution / Broker Integration
Execution هي الجهة الوحيدة التي تتعامل مع الوسيط لإرسال أو تعديل أو إلغاء الأوامر. لا تستقبل Proposal خاماً، بل Mission معتمدة فقط. مسؤولياتها:
*   إعادة التحقق من صلاحية المهمة وقت الإرسال.
*   تطبيع الأسعار والأحجام وفق خصائص الرمز.
*   فحص stops level وfreeze level وtick size وحالة السوق.
*   فحص نوع الحساب وقدرة نوع الأمر.
*   إرسال الأمر والتقاط النتيجة الفعلية.
*   إدارة حالات القبول والرفض والملء الجزئي وإعادة التسعير.
*   إلغاء الطرف المقابل في OCO وفق عقد محدد.
*   إنتاج ExecutionResult قابل للتدقيق.

**حد الملكية:** Risk يملك صلاحية المخاطرة، Execution يملك بروتوكول الوسيط. لا يجوز لـExecution توسيع المخاطرة لتجاوز قيود الوسيط.
## 1.6 Position Management
بعد نشوء مركز أو أمر حي، تنتقل الملكية التشغيلية إلى Position Management ضمن سياسة محددة. الأفعال المذكورة:
*   Hold.
*   Modify.
*   Partial Close.
*   Close.
*   Emergency Flat.

هذه الطبقة لا تعيد فتح قرار دخول جديد من تلقاء نفسها. كل تعديل يجب أن يكون مقيداً بسياسة، مرتبطاً بهوية Mission الأصلية، ومسجلاً في التدقيق.
## 1.7 Operations / Runtime / Lifecycle
هذه الطبقة تنظم دورة حياة النظام، لا منطق التداول نفسه:
*   warmup.
*   cadence على new bar.
*   dedupe.
*   health states.
*   recovery.
*   dashboard.
*   Telegram.
*   manual controls.

الداشبورد واجهة تفسير وتحكم، ويجب أن يعرض: حالة السوق، النوع المختار، الثقة، سبب المنع، حالة الحساب، الصفقات، المخاطر، أقرب دعم ومقاومة، والفصل بين H1 وM15.

الأزرار المطلوبة: TREND ON/OFF، OCO ON/OFF، ARM MANUAL OCO، CLOSE ALL، CANCEL PENDING، SHOW/HIDE LOG.

**حد الملكية:** الواجهة تطلب Commands ولا تعدّل state الداخلية مباشرة. Runtime/Command Handler يتحقق من الأمر ثم يمرره للمالك الصحيح.
## 1.8 Audit / Persistence / Replay
Audit مسار مستقل يستقبل Records من كل مرحلة:
*   Snapshot identity and health.
*   Decision and reason.
*   Proposal.
*   Risk approval/rejection.
*   Mission.
*   broker request/result.
*   OCO actions.
*   position management actions.
*   operator commands.
*   health/recovery transitions.

الهدف هو trace كامل، persistence، وإعادة تشغيل deterministic قدر الإمكان، مع schema parity بين backtest وlive.

**حد الملكية:** Audit لا يغيّر القرار ولا ينفذ أوامر. فشل الكتابة يجب أن يخضع لسياسة معلنة: هل يمنع التداول أم يضع النظام في degraded/halted state؟ هذا غير محسوم حالياً.

* * *
# 2\. فحص التماسك والعقود والملكية
## 2.1 ما هو متماسك مبدئياً
*   اتجاه البيانات أحادي وواضح: Reading → Context → Decision → Risk → Execution → Management → Audit.
*   فصل H1 للبنية عن M15 للمراقبة فكرة سليمة ولا تتعارض مع الفصل الطبقي.
*   Manual OCO لا يُفترض أن يتجاوز Risk أو Execution.
*   قرار «محرك واحد حسب النظام» يقلل تنافس الاستراتيجيات.
*   Snapshot المختومة تدعم العزل وإعادة التشغيل.
*   منع upsizing يتوافق مع fail closed.
*   الفصل بين Proposal وApproved Mission قرار معماري صحيح.
## 2.2 التناقضات أو نقاط عدم الاتساق
### C-01: لا يوجد فهرس قرارات فعلي
المستند يشير إلى «فهرس القرارات»، لكنه غير مرفق. لذلك لا يمكن مقارنة القرارات أو كشف تعارضات بين ADRs بصورة موثوقة.

**التصنيف:** Missing Specification.
**الأثر:** أي حكم على الاتساق بين القرارات سيكون تخميناً.
### C-02: معنى MANUAL غير مستقر
ورد أن MANUAL هو Operating Mode فقط، وورد أيضاً أن Manual OCO يأتي كطلب مستخدم داخل نفس الخط، ثم ظهرت عبارة تاريخية تقول إن المقترحات التلقائية في MANUAL تصبح تنبيهات فقط. لا يوجد عقد رسمي يحدد:
*   هل Manual Mode يعطل التنفيذ التلقائي كلياً؟
*   هل ARM MANUAL OCO طلب one-shot أم حالة مستمرة؟
*   هل TREND/OCO ON/OFF سياسات أهلية أم أوامر تنفيذ؟

**التصنيف:** Ambiguous Specification.
**الأثر:** خطر تنفيذ تلقائي غير مقصود أو منع صفقات مقصودة.
### C-03: مرجع قياس المسافة متعارض
إحدى العبارات تستخدم إغلاق آخر شمعة M15، وأخرى تستخدم السعر الحالي. كما أن الدعم والمقاومة قد يكونان مستويات Fibonacci فقط أو كل المستويات البنيوية.

**التصنيف:** Ambiguous Specification.
**الأثر:** اختلاف الداشبورد، triggers، ونتائج الاختبار الحي عن الباكتيست.
### C-04: ترتيب Policy / Reaction / Structure غير محسوم
الرسم البديل يضع Regime → Policy → Reaction → Structure، بينما الوصف الأساسي يجعل Structure من Market Reading قبل Policy. إذا كانت Structure حقيقة سوق، فلا ينبغي أن تأتي بعد اختيار السياسة.

**التصنيف:** Architectural Conflict.
**الأثر:** احتمال أن تغيّر السياسة طريقة إنتاج facts، ما يكسر canonical context.

**Proposal، تغيير معماري:** تثبيت Structure وReaction facts داخل Market Reading، ثم Policy بعدها. إذا كان Reaction يعني trigger تكتيكي، يُسمى Operational Trigger ويظل fact قبل القرار.
### C-05: ملكية OCO موزعة
Execution مسؤول عن OCO cancellation، لكن Position Management يملك lifecycle للأوامر والمراكز. لا يوجد حد واضح لمن يراقب fill ومن يرسل cancel ومن يعالج السباق.

**التصنيف:** Ambiguous Ownership.
**الأثر:** double cancel، orphan pending order، أو فتح الطرفين عند السباق.
### C-06: «Snapshot غير قابلة للتعديل» ليست عقداً تقنياً بعد
MQL5 لا يوفر immutability عميقة تلقائية. إذا احتوت Snapshot على references أو arrays أو handles قابلة للتغيير، يمكن كسر الختم.

**التصنيف:** Implementation Detail Missing.
**الأثر:** replay غير موثوق وقرارات لا يمكن تفسيرها.
### C-07: Position Management مذكورة بلا سياسة مصدر
لا نعرف من يصدر Modify/Partial Close/Close: سياسة ثابتة، محرك الدخول، Risk، أم Manager مستقل.

**التصنيف:** Missing Specification.
**الأثر:** كسر single writer على المركز، وتعارض trailing stop مع emergency logic.
### C-08: Score وConfidence وValidator متداخلة
كل محرك يحتوي Score/Confidence/Risk، ثم توجد Risk Layer مستقلة وValidator في رسم آخر. إذا احتوى المحرك على Risk حقيقي، فهذا يكرر الملكية.

**التصنيف:** Ambiguous Specification.
**الأثر:** قواعد مخاطرة متعارضة أو قبول مختلف لنفس المقترح.

**توضيح معماري:** ما داخل المحرك يجب أن يكون Risk Hints أو trade geometry، لا صلاحية مخاطرة ولا sizing نهائي.
## 2.3 مراجعة single writer
الوضع المقترح الصحيح، لكنه غير مثبت بعقود بعد:
*   Market readers تكتب facts الأولية، وContext Builder وحده يكتب Snapshot النهائية.
*   Decision Authority وحدها تكتب Decision النهائي.
*   Engine المختار يكتب Proposal واحداً تحت طلب Authority.
*   Risk وحدها تكتب approval وsizing النهائي.
*   Execution وحدها ترسل broker commands.
*   Position Manager وحده يطلب تعديلات المراكز بعد الفتح.
*   Runtime وحده يغير health/mode state.
*   Audit sink يضيف records فقط ولا يعدل السجل.

**الكسر المحتمل حالياً:** OCO cancellation، manual commands، position modifications، وRisk داخل المحركات. هذه تحتاج تعيين مالك صريح قبل التنفيذ.

* * *
# 3\. الثغرات قبل التنفيذ
## 3.1 Missing Specification
### MS-01: قاموس الأنواع والقيم
لا توجد enums أو حالات قانونية لـRegime وTrend وStructure وReaction وHealth وDecision وMission وExecutionResult.

**الأثر:** اختلاف تفسير الوحدات للقيمة نفسها.
### MS-02: عقود readers
لا توجد مدخلات ومخرجات وشروط صلاحية لكل Reader، ولا كيفية دمج تعارضاتها.

**الأثر:** Context ناقصة أو مزدوجة المصدر.
### MS-03: تعريف swing المعتبر
لا يوجد fractal depth، minimum bars، tie handling، confirmation delay، أو اختيار الموجة عند تعدد swings.

**الأثر:** Fibonacci وS/R غير قابلين لإعادة الإنتاج.
### MS-04: اتجاه Fibonacci والمستويات
لا توجد قاعدة anchoring للاتجاه، ولا قائمة مستويات نهائية، ولا تعريف support/resistance عند تداخل مستويات.

**الأثر:** عكس الشبكة أو اختلاف المنطقة بين trend صاعد وهابط.
### MS-05: تعريف منطقة الاهتمام
هل هي مستوى نقطي، نطاق حول المستوى، cluster، أم zone مبنية على ATR/ticks؟

**الأثر:** لا يمكن تعريف الاقتراب أو التأكيد على M15.
### MS-06: مصفوفة Regime → Engine → Trade Type
لا توجد truth table تحدد متى Trend، Breakout، Reversal، أو No Trade.

**الأثر:** Arbiter سيحتاج تخميناً.
### MS-07: معادلة Score وConfidence
لا توجد features، weights، normalization، نطاق، thresholds، أو معنى احتمالي.

**الأثر:** لا يمكن اختبار الحد الأدنى للثقة.
### MS-08: تعريف RR
هل يُحسب قبل spread/commission/slippage أم بعدها؟ وهل TP ثابت أو متعدد؟

**الأثر:** قبول صفقات لا تحقق العائد المطلوب فعلياً.
### MS-09: نموذج المخاطرة والحجم
ينقص: risk per trade، equity/balance/free margin basis، tick value handling، rounding، min/max/step volume، وحدود الخسارة اليومية.

**الأثر:** sizing خاطئ، وهو أخطر موضع في نظام مرفوع مالياً.
### MS-10: حدود المحفظة والتعرض
لا توجد قواعد للتعرض حسب الرمز، العملة، الأصل، الارتباط، الاتجاه، وعدد الصفقات.

**الأثر:** تضاعف خطر مخفي عبر أزواج مترابطة.
### MS-11: lifecycle للمهمة
لا توجد حالات mission ولا transitions ولا TTL ولا قواعد invalidation بعد تغير السوق.

**الأثر:** إرسال قرار قديم بعد تغير spread أو السعر أو regime.
### MS-12: عقد OCO
ينقص تعريف trigger، expiration، partial fill، simultaneous fill، cancel retry، reconnect، وما إذا كان OCO broker-native أو synthetic.

**الأثر:** انكشاف بطرفين بدلاً من طرف واحد.
### MS-13: سياسة Position Management
لا توجد قواعد break-even، trailing، partials، modify frequency، أو emergency hierarchy.

**الأثر:** ملكية وسلوك غير محددين بعد الدخول.
### MS-14: حالة Runtime
لا توجد state machine لـBoot/Warmup/Ready/Degraded/Halted/Recovering/Shutdown.

**الأثر:** النظام قد يتداول أثناء نقص بيانات أو بعد reconnect غير مكتمل.
### MS-15: سياسة persistence والاستعادة
لا نعرف ما الذي يُحفظ، متى، صيغة versioning، وكيفية reconciliation مع الوسيط بعد إعادة التشغيل.

**الأثر:** تكرار أوامر أو فقد ربط الصفقة بمهمتها.
### MS-16: schema التدقيق
لا يوجد record envelope، correlation IDs، timestamps، version، reason codes، أو retention.

**الأثر:** replay وتحليل الحوادث غير مضمونين.
## 3.2 Ambiguous Specification
### AS-01: current price
Bid أم Ask أم Mid أم Last أم M15 close؟ ويجب أن يختلف المرجع أحياناً حسب جهة الأمر.
### AS-02: point مقابل pip مقابل tick
المستند يستخدم points وpips بالتبادل. في MQL5 ليست مترادفات.
### AS-03: new-bar cadence
هل القرار مرة كل M15 bar فقط، بينما broker/position events فورية؟ وهل H1 يتحدث فقط عند إغلاق شمعة؟
### AS-04: closed bar مقابل forming bar
لا يوجد قرار بشأن استخدام bar 0 أو bar 1 لكل reader.
### AS-05: session/timezone
لا توجد منطقة زمنية أو DST أو تعريف جلسات الفوركس/الذهب/المؤشرات.
### AS-06: No Trade مقابل Wait مقابل Disabled مقابل Halted
هذه حالات مختلفة تشغيلياً لكنها غير مفصولة.
### AS-07: Manual inputs
هل القيم المدخلة constraints أم overrides أم مجرد preferences؟
### AS-08: CLOSE ALL
هل يشمل الرمز الحالي، استراتيجية FOX فقط، كل symbols، pending orders، ومراكز يدوية؟
### AS-09: CANCEL PENDING
النطاق والملكية وmagic number غير محددة.
### AS-10: backtest/live parity
هل المطلوب تطابق schema فقط أم تطابق مسار القرار والأحداث أيضاً؟
## 3.3 Documentation Clarification
### DC-01
يجب فصل المتطلبات المعيارية عن سجل المحادثة والادعاءات السابقة بالتنفيذ.
### DC-02
يجب تعريف المصطلحات: Engine، Policy، Arbiter، Authority، Validator، Mission، Proposal، Reaction.
### DC-03
يجب وضع كل قاعدة بصيغة MUST/SHALL أو SHOULD، لا بصيغة وصفية عامة.
### DC-04
عبارة «H1 أدق» هي تفضيل تداولي، وليست حقيقة عامة. العقد الصحيح: H1 هو الفريم المعتمد لهذا workflow.
### DC-05
يجب توثيق أن H4 غير مستخدم افتراضياً، وهل هو ممنوع تماماً أم قابل للتهيئة مستقبلاً.
## 3.4 Implementation Detail Missing
### ID-01
آلية الختم في MQL5: private fields، copy semantics، وعدم تسريب arrays/handles قابلة للتعديل.
### ID-02
إدارة indicator handles: الإنشاء، BarsCalculated، CopyBuffer، release، وإعادة الإنشاء بعد تغير الرمز/الفريم.
### ID-03
مصدر الوقت: server time، local time، tester time، وترتيب الأحداث ذات الطابع الزمني نفسه.
### ID-04
تطبيع السعر والحجم وفق tick size وvolume step، لا digits فقط.
### ID-05
التعامل مع retcodes وtimeouts وrequotes والملء الجزئي.
### ID-06
idempotency/dedupe عبر ticks، new bars، restart، وtrade transaction events.
### ID-07
reconciliation بين state المحلية والأوامر/المراكز الحية لدى الوسيط.
### ID-08
حقن seams في MQL5 دون الاعتماد على reflection أو dynamic mocking غير المتاح.

* * *
# 4\. القرارات المفتوحة اللازمة قبل البدء
أقترح إنشاء Decision Register رسمي. هذا **Proposal توثيقي**، لا يغير المنطق.
## D-001: مرجع السعر والمسافة
يلزم تحديد Bid/Ask/Mid/Last/M15 close لكل عرض وحساب وtrigger، ووحدة العرض Points/Pips/Ticks.

**يغلق قبل:** FibonacciReader، dashboard، triggers، tests.
## D-002: تعريف swing وFibonacci
يلزم depth، confirmation، direction، levels، tie rules، invalidation.

**يغلق قبل:** StructureReader وFibonacciReader.
## D-003: تعريف S/R وZone of Interest
يلزم مصادر المستويات، أولوية المصادر، clustering، zone width، nearest selection.

**يغلق قبل:** Context schema وM15 monitoring.
## D-004: توقيت البيانات
يلزم bar 0/1، cadence، stale thresholds، H1/M15 alignment، gaps، session timezone.

**يغلق قبل:** جميع readers وreplay.
## D-005: نموذج صحة Snapshot
يلزم health states، required facts، optional facts، freshness، سبب invalidity.

**يغلق قبل:** Context sealing وDecision Authority.
## D-006: مصفوفة القرار
يلزم جدول Regime/Structure/Reaction/Session → engine/type/no-trade، مع tie-break وprecedence.

**يغلق قبل:** Decision Authority والمحركات.
## D-007: Score/Confidence
يلزم الصيغ والنطاقات والحدود والمعنى، وهل confidence calibrated أم heuristic.

**يغلق قبل:** Engine validation وRisk gates.
## D-008: Manual Mode وManual OCO
يلزم state machine للأوضاع، arm/disarm، one-shot، expiry، نطاق overrides، وسلوك auto proposals.

**يغلق قبل:** dashboard وDecision وExecution.
## D-009: نموذج Risk/Sizing
يلزم قاعدة رأس المال، المخاطرة، fees/slippage، SL geometry، RR، rounding، margin، daily limits.

**يغلق قبل:** Risk Layer وأي live execution.
## D-010: Exposure/Portfolio
يلزم limits حسب symbol/asset/currency/correlation واتجاه التعرض.

**يغلق قبل:** portfolio approval.
## D-011: Mission lifecycle
يلزم ID، states، TTL، invalidation، dedupe، one mission per decision cycle.

**يغلق قبل:** Execution.
## D-012: OCO semantics
يلزم order types، synthetic/native، race handling، partial fills، cancellation retries، restart behavior.

**يغلق قبل:** OCO Engine وExecution.
## D-013: Hedging مقابل Netting
يلزم الأنماط المدعومة وسياسة كل نمط، أو رفض التشغيل على غير المدعوم.

**يغلق قبل:** execution وposition management.
## D-014: Position Management Policy
يلزم المالك، الأفعال المسموحة، precedence، emergency rules، وعدم زيادة المخاطرة.

**يغلق قبل:** إدارة المراكز.
## D-015: Broker capability policy
يلزم allowed filling/order/expiration modes، stops/freeze behavior، market closure، min/max/step، error policy.

**يغلق قبل:** IBroker adapter.
## D-016: Runtime health/recovery
يلزم state machine، ما يمنع التداول، recovery gates، reconnect reconciliation، warmup.

**يغلق قبل:** التشغيل الحي.
## D-017: Audit failure policy
يلزم هل فشل IRecordSink يوقف التداول، وما الحد الأدنى للسجل الإلزامي، والـbuffering.

**يغلق قبل:** الربط النهائي والاعتماد الحي.
## D-018: نطاق أوامر الداشبورد
يلزم scope لـCLOSE ALL وCANCEL PENDING، permissions، confirmation، audit.

**يغلق قبل:** UI controls.
## D-019: معيار replay parity
يلزم تعريف event model وترتيب الأحداث والدقة المقبولة بين tester/live.

**يغلق قبل:** قبول النظام.

* * *
# 5\. مخاطر التنفيذ الخاصة بـMQL5
## 5.1 Immutability
MQL5 لا يضمن deep immutability. `const` وحدها لا تكفي إذا مررنا references أو arrays أو handles قابلة للتغير.

**الخطر:** Snapshot تتغير بعد القرار.
**المطلوب قبل التنفيذ:** عقد نسخ وختم واضح، ومنع كشف mutable internals.
## 5.2 Interfaces وDependency Injection
يمكن استخدام abstract classes/interfaces بأسلوب محدود مقارنة بلغات أحدث، لكن الإنشاء والملكية وحقن التبعيات يحتاجان انضباطاً يدوياً.

**الخطر:** coupling مباشر مع `CTrade` والدوال العالمية يصعب الاختبار.
**Proposal، تغيير بنيوي محدود:** اجعل IBroker وIAccountInfo وIRecordSink حدوداً صريحة منذ البداية، لا wrappers متأخرة.
## 5.3 إدارة الذاكرة والملكية
المؤشرات والكائنات المنشأة ديناميكياً تتطلب delete واضحاً، والنسخ غير المقصود قد يسبب تسرباً أو double ownership.

**الخطر:** موارد عالقة وسلوك غير مستقر على التشغيل الطويل.
## 5.4 Indicator Handles
الـhandles ليست بيانات. يجب انتظار `BarsCalculated`، والتحقق من `CopyBuffer`، ومعالجة نقص التاريخ، ثم `IndicatorRelease`.

**الخطر:** facts صفرية أو stale تبدو صحيحة نوعياً.
## 5.5 Event Model
`OnTick` و`OnTimer` و`OnTradeTransaction` قد تنتج تسلسلات تشغيلية متداخلة منطقياً. التنفيذ ليس multithreaded بالمعنى المعتاد، لكن reentrancy المنطقية وترتيب الأحداث ما زالا خطرين.

**الخطر:** إرسال مكرر أو OCO race أو state قديمة.
## 5.6 Tick Size وDigits وPoint
`NormalizeDouble(price, _Digits)` لا يكفي دائماً. يجب التطبيع إلى tick size. كذلك tick value قد يختلف حسب profit/loss direction أو symbol calculation mode.

**الخطر:** رفض وسيط أو sizing خاطئ.
## 5.7 Stops Level وFreeze Level
هذه القيود قد تختلف حسب الرمز والوسيط والحالة. Freeze level مهم خصوصاً للتعديل والإلغاء قرب السوق.

**الخطر:** مهمة مقبولة مخاطرياً لكنها غير قابلة للتنفيذ، أو عدم القدرة على تعديل الحماية.
## 5.8 Filling وOrder Modes
ليست كل الرموز تدعم كل أنواع الأوامر أو filling policies أو expiration modes.

**الخطر:** OCO Reversal/Breakout يعمل في الاختبار ويفشل حياً.
## 5.9 Hedging مقابل Netting
في netting يوجد مركز مجمع لكل رمز، بينما hedging يسمح بعدة مراكز. partial close، reverse، وربط المركز بالمهمة تختلف جذرياً.

**الخطر:** إغلاق أو تعديل exposure لا تخص المهمة المقصودة.
## 5.10 Trade Retcodes والملء الجزئي
نجاح استدعاء الإرسال لا يعني أن النتيجة التجارية النهائية نجحت. يلزم قراءة retcode وtransactions الفعلية.

**الخطر:** state محلية تقول Filled بينما الوسيط رفض أو ملأ جزئياً.
## 5.11 Strategy Tester مقابل Live
الـtester لا يحاكي دائماً latency، slippage، gaps، queueing، partial fills، freeze behavior، أو broker quirks بدقة.

**الخطر:** ثقة زائفة في replay parity.
## 5.12 Persistence وFile I/O
مسارات الملفات وقيود sandbox والتزامن وصلاحيات common files تختلف بين live/tester/agents.

**الخطر:** Audit أو recovery يعمل محلياً ويفشل في agents أو VPS.
## 5.13 UI Objects والداشبورد
Chart events والأزرار يمكن أن تولد clicks مكررة، وتتأثر بإعادة التهيئة وتغير chart/template.

**الخطر:** أمر manual مكرر أو فقد حالة arm بعد restart.
## 5.14 Telegram/Network
WebRequest يتطلب allowlisting وقد يفشل أو يحجب. لا ينبغي أن يكون الإخطار الخارجي داخل المسار الحرج للقرار.

**توضيح معماري:** Telegram sink مراقبة فقط، وليس مصدراً للحقيقة أو مالكاً للأوامر.

* * *
# 6\. خطة التنفيذ المقترحة: ترتيب القراءة = ترتيب البناء
## المرحلة 0: تثبيت العقود والقرارات
**المخرجات:** glossary، Decision Register، enums/state machines، data contracts، ownership table، reason codes.

**Definition of Done:**
*   أُغلقت D-001 إلى D-019 أو وُسمت صراحة بأنها خارج النطاق.
*   لكل field وحدة ومصدر وfreshness وقيمة invalid.
*   لكل state transitions قانونية.
*   لا توجد مسؤولية لها أكثر من مالك واحد.
*   لا كود تداول قبل اعتماد هذه الحزمة.
## المرحلة 1: Market Data وMarket Reading
البناء بالترتيب: مصدر الأسعار → time alignment → Structure H1 → Fibonacci H1 → S/R/zone → بقية readers → M15 operational observations.

**Seams الأساسية:**
*   **Proposal، إضافة seam:** IMarketData لتغذية bars/ticks محددة في الاختبار.
*   **Proposal، إضافة seam:** IClock لفصل server/test time.

**Definition of Done:**
*   كل Reader deterministic لنفس المدخل.
*   لا Reader يرسل أوامر أو يقرأ الحساب.
*   warmup ونقص البيانات وstaleness تنتج invalid reason صريحاً.
*   اختبارات golden cases للسوينغ، Fibonacci، nearest S/R، واتساق H1/M15.
*   لا استخدام ضمني لـH4.
## المرحلة 2: TradingContext
البناء: Builder → validation → seal → identity/version.

**Definition of Done:**
*   Snapshot لا تتغير بعد النشر، بما في ذلك arrays والحقول المركبة.
*   تحتوي structure TF وmonitor TF وأزمنة البيانات وhealth reasons.
*   لا facts مكررة بمالكين مختلفين.
*   serialization round-trip يحافظ على المعنى.
*   invalid context يمنع القرار.
## المرحلة 3: Decision Authority والمحركات
البناء: Policy matrix → Arbiter → Trend Engine → OCO Engine → Manual request adapter → NoTrade reasons.

**Definition of Done:**
*   محرك واحد كحد أقصى لكل cycle.
*   كل قرار قابل للتفسير من Snapshot وDecision Register.
*   المحرك لا يحسب sizing نهائي ولا يتصل بالوسيط.
*   manual request يمر بنفس authority ولا ينتج Mission مباشرة.
*   التعارض أو نقص fact ينتج No Trade، لا fallback تخميني.
## المرحلة 4: Risk / Sizing / Portfolio
**Seam إلزامية:** **IAccountInfo** لتمثيل balance/equity/free margin/positions/exposure دون دوال عالمية في الاختبار.

**Definition of Done:**
*   معادلات الحجم مختبرة على FX والذهب والمؤشرات واختلاف digits/tick values.
*   requested lot لا يزيد الحجم المسموح.
*   RR والتكاليف وslippage policy محسومة.
*   exposure limits وdaily halt مختبرة.
*   كل رفض يحمل reason code ومدخلات قابلة للتدقيق.
*   Approved Mission ثابتة ومكتملة ولها TTL.
## المرحلة 5: Execution وBroker Integration
**Seam إلزامية:** **IBroker** للأوامر والخصائص والنتائج وtrade events.

**Definition of Done:**
*   لا إرسال بلا Approved Mission صالحة.
*   التطبيع على tick size وvolume step.
*   دعم أو رفض صريح لكل capability ونمط حساب.
*   اختبارات stops/freeze/filling/market closed/requote/partial fill.
*   idempotency تمنع الإرسال المكرر.
*   OCO race وcancel retry وrestart reconciliation مختبرة.
*   Execution لا يزيد المخاطرة عند معالجة قيود الوسيط.
## المرحلة 6: Position Management
**Definition of Done:**
*   Position Manager هو single writer لطلبات التعديل بعد الفتح.
*   كل action مرتبط بمهمة ومركز/أمر محدد.
*   precedence بين emergency/close/modify/partial واضح.
*   لا تعديل يوسع SL أو exposure إلا إذا نص العقد صراحة، والأفضل منعه.
*   hedging/netting scenarios مختبرة منفصلة.
## المرحلة 7: Operations / Runtime / Dashboard
**Definition of Done:**
*   state machine كاملة لـwarmup/ready/degraded/halted/recovery.
*   new-bar decision cadence منفصلة عن مراقبة broker events.
*   dashboard يعرض facts والقرار وأسباب المنع دون إعادة حسابها.
*   الأزرار ترسل Commands مدققة، ولا تعدل state مباشرة.
*   ARM MANUAL OCO وCLOSE ALL وCANCEL PENDING لها نطاق وTTL وdedupe وتأكيد وفق القرار المعتمد.
*   فشل Telegram لا يوقف thread التداول ولا يغير القرار.
## المرحلة 8: Audit / Persistence / Replay
**Seam إلزامية:** **IRecordSink** لاستقبال سجلات موحدة في live/test/replay.

**Definition of Done:**
*   correlation واحد يربط Snapshot → Decision → Proposal → Risk → Mission → Execution → Position actions.
*   schema versioned ومتطابقة منطقياً بين backtest وlive.
*   ترتيب الأحداث deterministic أو موثق عند عدم إمكانه.
*   restart يعيد reconciliation دون duplicate order.
*   فشل sink يتبع policy معلنة ومختبرة.
*   replay يعيد نفس القرارات من نفس facts، مع توثيق الفروق التي يسببها broker simulation.
## المرحلة 9: Acceptance Gate قبل Live
**Definition of Done:**
*   اختبارات unit وcontract وscenario وfault injection ناجحة.
*   Walk-forward وout-of-sample لا يستخدمان لتغيير العقود بصمت.
*   dry-run/shadow mode يثبت القرار والتدقيق دون إرسال.
*   broker capability matrix للرموز المستهدفة مكتملة.
*   limits محافظة ومثبتة في الإعدادات.
*   مراجعة مستقلة لمسار sizing وemergency controls.

* * *
# Proposals التحسينية فقط، غير مطبقة
## P-01: فصل Trade Geometry عن Risk Authority
اجعل المحرك يقترح entry geometry وinvalidation/targets، بينما Risk وحدها تحولها إلى SL/TP/volume نهائي.

**السبب:** يمنع ازدواج ملكية المخاطرة.
**الأثر:** تعديل عقود Proposal وMission فقط، وتحسن الاختبارات والتدقيق.
## P-02: إضافة Decision Envelope موحد
كل دورة تحمل cycle ID، snapshot ID، policy version، decision reason، engine selected.

**السبب:** تتبع كامل ومنع التكرار.
**الأثر:** يمتد إلى Audit وMission وExecutionResult.
## P-03: جعل OCO Coordinator مكوّناً صريحاً داخل Execution domain
يتابع الطرفين والأحداث والإلغاء، بينما IBroker ينفذ فقط.

**السبب:** OCO بروتوكول ذو state وليس مجرد send/cancel.
**الأثر:** يزيل التداخل مع Position Management ويحتاج state machine واختبارات race.
## P-04: إضافة IMarketData وIClock بجانب seams المطلوبة
**السبب:** من دونهما يصعب اختبار H1/M15 وstaleness وreplay دون الاعتماد على بيئة المنصة.
**الأثر:** زيادة بسيطة في البنية، وفائدة كبيرة في determinism.
## P-05: فصل Dashboard View Model عن TradingContext
**السبب:** الواجهة تحتاج صياغة وعرضاً، لكنها لا يجب أن تعيد تفسير الحقائق.
**الأثر:** mapper للعرض فقط، ويمنع تسرب UI إلى القرار.
## P-06: اعتماد Contract Tests على كل seam
**السبب:** mock قد ينجح بينما adapter الحقيقي يخالف الدلالات.
**الأثر:** نفس سيناريوهات IBroker/IAccountInfo/IRecordSink تعمل على fake وعلى adapter الحقيقي حيث أمكن.

* * *
# الحكم النهائي
المعمارية الاتجاهية **جيدة وقابلة للبناء**: فصل القراءة عن القرار، Snapshot مختومة، سلطة قرار واحدة، Risk قبل Execution، وAudit شامل هي اختيارات صحيحة. لكن المستند الحالي **ليس مواصفة تنفيذية مكتملة**؛ هو blueprint أولي ممزوج بسجل محادثة وادعاءات تنفيذ غير قابلة للتحقق.

أكبر نقاط الإيقاف قبل كتابة أي كود هي: تعريف swing/Fibonacci والمسافة، مصفوفة القرار، معنى Manual Mode، نموذج sizing/exposure، lifecycle للمهمة، semantics الكاملة لـOCO، hedging/netting، وملكية Position Management.

**القرار المقترح:** لا يبدأ التنفيذ بعد. يُعتمد أولاً هذا الاستعراض، ثم نغلق D-001 إلى D-019 في Decision Register، وبعدها نثبت العقود ونبني حسب المراحل أعلاه. لم أكتب أو أعدّل أي كود.