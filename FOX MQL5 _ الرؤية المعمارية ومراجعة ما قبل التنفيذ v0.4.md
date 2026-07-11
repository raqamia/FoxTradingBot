# FOX MQL5 | الرؤية المعمارية ومراجعة ما قبل التنفيذ v0.4

# FOX MQL5 | الرؤية المعمارية ومراجعة ما قبل التنفيذ v0.4
**الحالة:** Vision Baseline + Review Gate، لا تنفيذ ولا كود
**المصادر التي تمت مراجعتها:** `arc.md` + توضيح المالك حول المحركات ووضعَي AUTO/MANUAL
**قاعدة المراجعة:** Fail closed، أي فراغ يُسجل ولا يُفترض
> **تحديث v0.4:** تم فصل سبب الدخول عن وسيلة الدخول وعن تشكيل الأوامر المعلقة: Opportunity تجيب لماذا، Entry Method تجيب كيف، وPending Entry Policy تحدد تكوين الـ Pending. AUTO وMANUAL يحددان صاحب القرار ولا يغيران هذا التصنيف.  
> **قيد مهم:** المستند المرفق يحتوي على وصف معماري، ومعه عبارات سابقة تزعم أن تغييرات نُفذت في الكود. لم تُرفق ملفات MQL5 أو Decision Index مستقل أو schemas أو اختبارات. لذلك أستطيع مراجعة الفكرة والعقود المكتوبة، لكن لا أستطيع التصديق على أن الفصل H1/M15 أو Manual OCO أو أي ضمان آخر “مثبت في الكود”. كل ادعاء تنفيذي في `arc.md` يُعامل كمرجع نية، لا كدليل تنفيذ.
* * *
# الخلاصة التنفيذية
الاتجاه العام **سليم وقابل للتحويل إلى منصة مؤسسية**: Market Intelligence تنتج حقائق، Context مختوم يجمدها، Decision Authority تختار مساراً واحداً، Risk تمنح أو ترفض التفويض، Execution تتعامل مع الوسيط، ثم Position Management وOperations وAudit.

لكن المعمارية **ليست جاهزة للبرمجة بعد**. توجد أربعة حواجز رئيسية:

1. لا توجد عقود بيانات مكتملة بين الطبقات: أنواع الحقول، الوحدات، الصحة، timestamps، الهوية والإصدارات.
2. سياسة التحكيم بين Trend وBreakout وReversal وManual OCO غير معرفة بصورة حتمية.
3. الفصل H1/M15 واضح كنية، لكنه غير محسوم حسابياً: هل مسافة المستوى تُقاس من live quote، أم إغلاق M15، أم إغلاق H1؟
4. حدود الملكية غير مكتملة عند Context construction، Risk approval، OCO cancellation، Position Management، وأوامر التشغيل اليدوية.

**قرار المراجعة الحالي: NOT READY FOR IMPLEMENTATION.** المطلوب إغلاق القرارات المفتوحة في القسم الرابع، ثم إصدار Architecture Baseline مع Decision Index وData Contracts قبل كتابة الكود.

## الرؤية التشغيلية المعتمدة من المالك
**\[توضيح للمعمارية\]** دورة FOX الرسمية:

Market Data → Market Intelligence → Sealed TradingContext → Market Classifier → Authority Gate → Engine → Risk/Safety Gate → Execution → Position Management → Closeout → Audit/Operations.

يصنف النظام السوق إلى نتيجة واحدة فقط لكل snapshot:
*   Trend
*   Breakout
*   Reversal
*   Wait

ثم يحدد مساراً واحداً:
*   Trend → Trend Engine.
*   Breakout أو Reversal → OCO Engine.
*   Wait → لا proposal ولا تنفيذ.

التصنيف الرسمي من ثلاث مراحل مستقلة:

1. `Opportunity`: لماذا سندخل؟ `Trend / Breakout / Reversal`.
2. `Entry Method`: كيف سندخل؟ `Market / Pending`.
3. `Pending Entry Policy`: إذا كان الدخول Pending، كيف تُنشأ الأوامر؟ `Two Side / One Side`.

عند `One Side` يلزم اتجاه صريح: `Buy Only / Sell Only`. عند `Two Side` فقط يوجد OCO حقيقي: أمران معلقان، وأول fill مؤكد يطلق إلغاء الطرف الآخر. هذا الفصل يمنع استعمال Breakout/Reversal مرة كسبب للفرصة ومرة كسياسة لإنشاء الأوامر.

AUTO وMANUAL يحددان **من يملك قرار الدخول**:
*   **AUTO:** النظام يصنف، يختار المسار، يبني المهمة ويجيز تنفيذها بعد كل البوابات.
*   **MANUAL:** النظام يقرأ ويعرض الفرصة، لكنه يتوقف عند `Awaiting Human Decision`. لا ينشئ دخولاً تنفيذياً حتى يسلح المتداول طلباً صريحاً.

في MANUAL تكون قيم `fixed lot / SL / TP` مدخلات بشرية ملزمة للطلب، خصوصاً في صفقات OCO/scalping ذات اللوت الأعلى والمسافات القصيرة. لكنها لا تتجاوز سلامة الوسيط ولا الغلاف الصلب للحساب.

**\[تغيير معماري مقترح P-08\]** فصل بوابتين:

1. `Decision Authority`: آلية في AUTO وبشرية عبر Manual Command في MANUAL.
2. `Hard Safety Authority`: آلية دائماً ولا يمكن تعطيلها، وتتحقق من صلاحية السعر والحجم والهامش وقيود الوسيط والحدود الكارثية.

التبرير: احترام القرار البشري لا يعني السماح بأمر غير صالح أو بحجم يتجاوز سقف الحساب. في MANUAL لا تعيد Risk تحسين اللوت أو SL/TP ولا تستبدل رأي المتداول؛ إما تقبل القيم كما هي بعد التطبيع المحافظ، أو ترفضها بسبب واضح.

**الأثر:** يلزم تمييز `Risk Optimization` الخاص بـ AUTO عن `Hard Safety Validation` المشترك بين الوضعين.

* * *
# 1) فهم المعمارية، طبقةً طبقة
## 1.1 Market Reading / Market Intelligence
**\[توضيح للمعمارية الحالية\]** هذه الطبقة تقرأ السوق فقط ولا تقرر التداول ولا تحسب اللوت ولا ترسل أمراً. مكوناتها المقترحة:
*   Trend
*   Regime
*   Structure / Location
*   Liquidity
*   Momentum
*   Volatility
*   Session
*   SMC Blocks
*   Fibonacci

الفصل الزمني المقصود هو:
*   **H1:** البنية المرجعية، آخر swing معتبر، Fibonacci، الدعم والمقاومة الأقرب.
*   **M15:** المراقبة التشغيلية للسلوك قرب المستويات: اقتراب، تأكيد، اختراق، ارتداد، أو غياب edge.
*   **H4:** ليس الإطار الافتراضي لهذا workflow.

المخرجات يجب أن تكون **حقائق canonical typed facts**، لا توصيات تداول. مثال مفاهيمي: اتجاه H1، regime، swing identity، مستويات السعر، المسافات بوحدة معلنة، ATR، spread، session، freshness، وجود/غياب البيانات.

**حد الملكية:** كل Reader يملك حساب مخرجه فقط. لا يحق له اختيار Engine أو إنشاء صفقة. ما يزال غير محدد من يدمج هذه المخرجات في snapshot واحدة.
## 1.2 TradingContext / Shared State
**\[توضيح للمعمارية الحالية\]** TradingContext هي snapshot واحدة مختومة لكل دورة قرار، وتحتوي:
*   الحقائق المعيارية الناتجة عن القرّاء.
*   قيماً typed، لا سلاسل نصية حرة.
*   حالة صحة snapshot.
*   structure timeframe وmonitor timeframe.
*   هوية زمنية تمنع خلط بيانات من دورات أو شموع مختلفة.

بعد الختم تكون read-only من منظور الطبقات التالية. Decision وRisk وExecution لا تعيد قراءة المؤشرات من السوق ولا تعدل Context.

**حد الملكية المطلوب:** ContextBuilder واحد هو الكاتب الوحيد. Readers يعيدون نتائج مستقلة، وContextBuilder يتحقق منها ويجمعها ويختمها. هذا الحد **غير مثبت صراحةً** في المستند الحالي.
## 1.3 Decision Authority
**\[توضيح للمعمارية الحالية\]** هذه هي سلطة القرار الوحيدة. تقرأ snapshot المختومة، تصنف فرصة التداول إلى:
*   Trend trade
*   Breakout trade
*   Reversal trade
*   No trade / Wait

ثم تختار **محركاً واحداً فقط** وفق single-engine-by-regime:
*   Trend Engine لصفقات الاتجاه.
*   OCO / Compression Engine للاختراق أو الارتداد.
*   Manual OCO هو طلب تشغيل يدوي، لكنه لا يتجاوز نفس المسار.

المحرك المختار ينتج TradeProposal واحدة، وليس أمراً للوسيط. Score وConfidence يجب أن يكونا معلومات قرار، أما Risk فلا ينبغي أن تكون “نسخة داخل كل Engine” إذا كانت هناك Risk Authority مركزية، إلا إن كانت Risk داخل المحرك مجرد risk hints غير ملزمة.

**حد الملكية:** Decision Authority وحدها تختار نوع الصفقة والمحرك وتصدر proposal. المحركات لا تنفذ ولا تمنح موافقة المخاطرة.
## 1.4 Risk / Sizing / Portfolio
**\[توضيح للمعمارية الحالية\]** تستقبل proposal وsnapshot وحالة الحساب/المحفظة، ثم تقوم بـ:
*   اشتقاق أو التحقق من SL وTP.
*   حساب الحجم.
*   التحقق من الحد الأدنى RR والثقة.
*   التحقق من exposure وحدود الحساب واليوم والرمز.
*   رفض أي manual requested lot يتجاوز الحجم المسموح.
*   Fail closed عند نقص المدخلات أو عدم صلاحيتها.

المخرج الصحيح معمارياً ليس proposal معدلة بصمت، بل **RiskDecision** صريحة: ApprovedMission أو Rejection مع reasons وأرقام الحساب المستخدمة.

**حد الملكية:** Risk Authority هي الكاتب الوحيد للحجم المعتمد، risk amount، SL/TP النهائية، وسبب الرفض. Decision لا تعتمد المهمة، وExecution لا تعيد sizing.
## 1.5 Execution / Broker Integration
**\[توضيح للمعمارية الحالية\]** لا تقبل إلا ApprovedMission صالحة وغير منتهية. مسؤولياتها:
*   قراءة قيود الرمز والحساب والوسيط.
*   تطبيع الأسعار والأحجام وفق tick size وvolume step.
*   فحص stops level وfreeze level ونوع الحساب.
*   إرسال الأوامر وتفسير retcodes.
*   إنتاج ExecutionResult.
*   إدارة lifecycle لأوامر OCO المعلقة وإلغاء الطرف الآخر وفق سياسة محددة.

**حد الملكية:** Execution تملك broker requests ونتائجها فقط. لا تعيد تصنيف السوق ولا تغير منطق القرار أو ميزانية المخاطر.
## 1.6 Position Management
**\[توضيح للمعمارية الحالية\]** تبدأ ملكيتها بعد وجود position أو pending-order lifecycle معترف به، وتشمل:
*   Hold
*   Modify
*   Partial close
*   Close
*   Emergency flat

ينبغي أن تعمل عبر أوامر سياسة واضحة، لا عبر تعديل عشوائي للصفقة. العلاقة الدقيقة بينها وبين OCO cancellation وExecution غير محسومة بعد.
## 1.7 Operations / Runtime / Lifecycle
**\[توضيح للمعمارية الحالية\]** تنسق التشغيل دون امتلاك قرار التداول:
*   Warmup
*   new-bar cadence
*   dedupe
*   health state
*   restart/recovery
*   dashboard
*   Telegram
*   manual commands

الداشبورد يعرض الحالة ولا يعيد حساب الحقيقة. الأزرار TREND ON/OFF وOCO ON/OFF وARM MANUAL OCO وCLOSE ALL وCANCEL PENDING وSHOW/HIDE LOG يجب أن تنتج **Control Commands** مدققة، لا تعديلات مباشرة في state الداخلية.
## 1.8 Audit / Persistence / Replay
**\[توضيح للمعمارية الحالية\]** تسجل سلسلة غير قابلة للالتباس من:

Market facts → Snapshot sealed → Decision/No-trade → Proposal → Risk approved/rejected → Broker request/result → Position action → Operator command.

Replay يجب أن يعيد تشغيل نفس inputs والإصدارات ويقارن المخرجات دون الوصول إلى وسيط حقيقي. parity بين live وbacktest تعني نفس schemas والمنطق، لا بالضرورة نفس adapter أو clock.

* * *
# 2) فحص التماسك والعقود والملكية
## 2.1 ما هو متماسك
*   فصل القراءة عن القرار عن المخاطر عن التنفيذ صحيح.
*   Context مختومة تمنع الطبقات اللاحقة من تغيير حقائق السوق.
*   “لا تنفيذ بلا mission approved” حد ممتاز.
*   Manual OCO يمر عبر Decision وRisk وExecution، وهذا يمنع bypass يدوي خطير.
*   H1 للبنية وM15 للمراقبة فصل مفهوم ومناسب، بشرط حسم سعر القياس ووحدة المسافة.
*   fail closed ورفض upsizing اليدوي متوافقان مع بيئة الرافعة.
*   Audit وReplay وschema parity موضوعة كجزء من النظام، لا كإضافة لاحقة.
## 2.2 التعارضات أو التوترات الحالية
### C-01: معنى “single-engine-by-regime” غير مكتمل
النص يقول إن regime يحدد محركاً واحداً، ويقول أيضاً إن Decision Authority تحدد نوع الصفقة ثم تختار المحرك. لا يوجد جدول حتمي يوضح أيهما سابق: regime أم trade type، ولا ما يحدث إذا أعطت القراءات إشارات Trend وBreakout معاً.

**الأثر:** احتمال تشغيل أكثر من محرك أو اختلاف live عن backtest.
### C-02: OCO Engine يجمع Breakout وReversal دون عقد فصل
Breakout عادةً يستخدم Stop orders، وReversal قد يستخدم Limit orders، والاثنان يختلفان جذرياً في شروط التأكيد والإلغاء وقياس الخطر. جمعهما ممكن، لكن يلزم policy typed واضحة.

**الأثر:** فرع واحد قد يخلط order semantics أو يلغي الطرف الخطأ.
### C-03: Manual كـ Operating Mode مقابل ARM MANUAL OCO
المستند يقول إن MANUAL Operating Mode فقط، ويذكر أيضاً Manual OCO request one-shot وInputs “MANUAL mode only”، بينما النص العام يوحي بأن المحركات الآلية تعمل وتصدر proposals. غير واضح هل MANUAL يمنع كل تنفيذ تلقائي، أم هو مجرد مصدر intent داخل وضع أوسع.

**الأثر:** خطر تنفيذ غير متوقع أو منع تنفيذ مقصود.
### C-04: Risk داخل كل Engine مقابل Risk Layer مركزية
الرسم يضع Score/Confidence/Risk تحت Trend Engine وOCO Engine، ثم يضع Risk Layer مستقلة. إن كانت “Risk” داخل المحرك سلطة sizing فهذه ازدواجية وكسر single writer. إن كانت مجرد metadata فيجب تسميتها بوضوح.

**الأثر:** حجمان أو SL/TP مختلفان لنفس proposal.
### C-05: سعر قياس مسافات H1/M15 متعارض دلالياً
ورد أن Fibonacci والمستويات من H1، وأن “سعر القياس” هو إغلاق M15، ثم جاء طلب أن “المعادلات مبنية على H1”. لا يمكن اعتبار الثلاثة شيئاً واحداً. Anchor timeframe شيء، والسعر الذي تُقاس منه المسافة شيء آخر.

**الأثر:** اختلاف Zone of Interest والقرارات عند كل شمعة أو tick.
### C-06: “عدم تشغيل الخمسة معاً” غير قابل للتحقق
لا توجد قائمة رسمية للخمس وحدات المقصودة. النص الرسمي يذكر محركين، بينما مقتطفات أخرى تشير إلى Sc...lping Model وMeta Allocator/Portfolio Manager.

**الأثر:** لا يمكن تصميم arbiter أو اختبار exclusivity.
### C-07: Scope التنفيذ الآلي غير واضح
عبارة سابقة داخل الملف تقول إن “المقترحات التلقائية تبقى تنبيهات فقط”، بينما المعمارية العامة تصف منصة تداول وتنفيذ بعد اعتماد المهمة. هل Trend/OCO الآليان ينفذان أم لا؟

**الأثر:** هذا قرار product safety أساسي، وليس implementation detail.
## 2.3 فحص single writer
*   **Market facts:** مهددة إذا كتب Readers مباشرة داخل Context. يجب ContextBuilder واحد.
*   **Snapshot health:** تحتاج مالكاً واحداً، والأفضل ContextBuilder/HealthPolicy.
*   **Trade classification / engine selection:** مالكها Decision Authority، وهذا متسق نظرياً.
*   **Proposal:** يجب أن يكتبها Engine المختار مرة واحدة، ثم لا تُعدل.
*   **Final risk values:** مهددة إذا عدل Engine أو Execution الحجم/SL/TP. يجب Risk Authority وحدها.
*   **Broker state:** Execution Adapter يكتب requests، لكن broker هو مصدر الحقيقة للقبول والملء.
*   **OCO lifecycle:** المالك غير محدد بين Execution وPosition Management وOperations.
*   **Open position lifecycle:** Position Management ينبغي أن تكون المالك الوحيد للسياسة، وExecution مجرد منفذ أوامر broker.
*   **Dashboard state:** يجب أن يكون projection read-only من events/snapshot، لا state موازية.
*   **Manual controls:** تحتاج Command Gateway واحداً مع audit وauthorization وdedupe.

**النتيجة:** single writer معلن كفكرة، لكنه غير مغلق تعاقدياً في خمس نقاط: Context، risk outputs، OCO lifecycle، position lifecycle، control commands.

* * *
# 3) الثغرات قبل التنفيذ
## 3.1 Missing Specification
1. **Data contracts:** لا schema كاملة لـ ReaderResult وTradingContext وTradeProposal وRiskDecision وApprovedMission وExecutionResult وPositionAction وAuditEvent.
2. **Time identity:** لا تعريف لـ snapshotId، symbol، server time، bar open time، source timeframe، data age، build version.
3. **Swing algorithm:** لا تعريف fractal depth، confirmation، tie handling، minimum span، orientation، invalidation، وعدد bars.
4. **Fibonacci contract:** المستويات الإلزامية، اتجاه الرسم، التعامل مع trend up/down، zero range، واختيار آخر swing “معتبر”.
5. **Support/resistance universe:** هل الأقرب من Fib فقط، أم swings وSMC وstructure zones أيضاً؟
6. **Distance units:** point أم pip أم tick أم price، وكيف تعرض الرموز ذات 3/5 digits والذهب والمؤشرات.
7. **Regime taxonomy:** القيم الممكنة وقواعد الانتقال والثبات/hysteresis.
8. **Decision matrix:** mapping حتمي من facts/regime إلى Trend/Breakout/Reversal/NoTrade والمحرك المختار.
9. **Score/confidence:** الصيغة، النطاق، calibration، thresholds، missing-feature behavior.
10. **Risk policy:** risk per trade، daily loss، max drawdown، max exposure، max trades، correlation/currency concentration، minimum RR/confidence.
11. **Cost model:** spread، commission، swap، slippage، gap risk، وهذه ضرورية لحساب RR الحقيقي.
12. **Order policy:** exact order types، expiry، filling policy، deviation، retries، partial fill، rejection handling.
13. **OCO state machine:** متى يعتبر طرفاً activated/filled، ومتى وكيف يلغى الآخر، وماذا عند fill متزامن أو فشل الإلغاء.
14. **Position management policy:** شروط modify/partial/close/trailing/breakeven/emergency.
15. **Modes and permissions:** AUTO/MANUAL/PAUSED/HALTED semantics، ومن يستطيع تشغيل الأزرار.
16. **Recovery:** مصدر الحقيقة بعد restart، ومصالحة الأوامر/المراكز orphaned.
17. **Audit schema:** event names، sequence، correlation IDs، versioning، retention، integrity.
18. **Replay contract:** clock، random-free behavior، input dataset، broker simulator، comparison tolerance.
19. **Portfolio manager:** مذكور كفكرة، بلا موقع أو عقد أو أولوية قرار.
20. **الأصول المدعومة:** Forex/Gold/Indices ذكرت ضمن السياق، لكن لا توجد capability matrix للرموز والوسطاء.
## 3.2 Ambiguous Specification
1. سعر قياس المسافة: Bid/Ask/Mid/last tick أم M15 close أم H1 close.
2. معنى “support” و“resistance” عند level مساوٍ للسعر أو داخل zone.
3. هل distances signed أم absolute، وماذا لو لا يوجد مستوى على أحد الجانبين.
4. M15 “شمعة تأكيد”: closed bar فقط أم current forming bar.
5. new-bar cadence: هل decision مرة كل M15، مع H1 refresh منفصل، أم tick-level proximity trigger.
6. MANUAL: operating mode عالمي أم request source أم permission gate.
7. TREND ON/OFF وOCO ON/OFF: هل تمنع القراءة، proposal، أم التنفيذ فقط.
8. CLOSE ALL: الرمز الحالي أم كل رموز FOX أم الحساب كله.
9. CANCEL PENDING: أوامر FOX فقط أم كل أوامر الحساب.
10. “confidence” هل هي احتمال calibrated أم score normalized.
11. “snapshot health” هل هي boolean أم enum متعدد الأسباب.
12. “schema parity”: parity بنيوية فقط أم bit-for-bit behavior.
13. “immutable audit”: append-only منطقي أم tamper-evident cryptographic chain.
## 3.3 Documentation Clarification
1. فصل النص المعماري الرسمي عن سجل محادثات أو ادعاءات “تم التنفيذ”.
2. توفير Decision Index حقيقي بأرقام ADR/DEC وروابط وتواريخ وحالة Accepted/Open/Superseded.
3. تعريف glossary رسمي: point، pip، tick، zone، swing، regime، proposal، mission، manual OCO.
4. توثيق dependency direction ومنع أي طبقة من القراءة للخلف.
5. توثيق أن dashboard projection فقط وليس authority.
6. توضيح إن كان Telegram outbound alerts فقط أم يقبل commands.
7. توضيح المقصود بـ “الخمسة” وScalping Model وMeta Allocator.
8. وضع non-goals للنسخة الأولى: multi-symbol، portfolio optimization، Telegram commands، advanced replay، وغيرها.
## 3.4 Implementation Detail Missing
1. تمثيل immutability في MQL5.
2. نمط interfaces/adapters والـ dependency injection العملي في MQL5.
3. ownership وrelease لكل indicator handle.
4. CopyBuffer/CopyRates readiness وBarsCalculated وwarmup.
5. price/volume normalization وفق tick size وvolume step، لا digits فقط.
6. broker retcode matrix وسياسة retry الآمنة.
7. idempotency/dedupe keys للأوامر والأحداث.
8. Magic Number وcomment naming وsymbol partitioning.
9. event loop بين OnInit/OnTick/OnTimer/OnTradeTransaction/OnChartEvent.
10. persistence format والكتابة الذرية وflush/failure behavior.
11. test doubles لـ IBroker وIAccountInfo وIRecordSink وIClock وIMarketData.
12. deterministic backtest clock ومعالجة اختلاف جودة tick data.
> لا يجوز “حل” البنود أعلاه تلقائياً أثناء البرمجة. كل بند يؤثر في السلوك أو المخاطر يجب أن يُغلق بقرار قبل implementation.
* * *
# 4) القرارات المفتوحة المطلوبة قبل البدء
## O-01: مصدر السعر لحساب المسافة
**يُغلق بتحديد:** هل القياس من executable quote (Ask للشراء/Bid للبيع)، Mid، آخر tick، آخر M15 close، أم H1 close؛ وهل العرض والقرار يستخدمان المصدر نفسه.
**تعتمد عليه:** nearest S/R، zone proximity، dashboard، triggers، replay.
## O-02: تعريف swing H1
**يُغلق بتحديد:** algorithm، depth، confirmation bars، minimum range، invalidation، tie policy، warmup.
**تعتمد عليه:** Structure وFibonacci وكل قرار location-based.
## O-03: universe المستويات
**يُغلق بتحديد:** هل nearest support/resistance من Fib high/low/ratios فقط، أم من structure/SMC أيضاً، وكيف ترتب zones المتداخلة.
**تعتمد عليه:** dashboard، proximity، reversal/breakout.
## O-04: الوحدات والتطبيع
**يُغلق بتحديد:** internal unit canonical لكل من price distance وrisk وvolume، وقواعد conversion للعرض.
**تعتمد عليه:** كل الحسابات متعددة الأصول والوسيط.
## O-05: Regime taxonomy وtransition policy
**يُغلق بتحديد:** regimes، thresholds، hysteresis، minimum persistence، UNKNOWN behavior.
**تعتمد عليه:** single-engine selection وNoTrade.
## O-06: Decision precedence matrix
**يُغلق بتحديد:** ترتيب Trend/Breakout/Reversal، conflict resolution، minimum evidence، ومنع multi-engine.
**تعتمد عليه:** Decision Authority والمحركات والاختبارات.
## O-07: تعريف المحركات الرسمية
**يُغلق بتحديد:** القائمة النهائية للمحركات في v1، وتفسير “الخمسة”، ومصير Scalping Model وMeta Allocator.
**تعتمد عليه:** interfaces، arbiter، dashboard، configuration.
## O-08: Auto vs Manual semantics
**يُغلق بتحديد:** modes، transitions، ما الذي يُنفذ فعلياً، one-shot behavior، timeout، re-arm، وصلاحيات المستخدم.
**تعتمد عليه:** UI، execution gate، audit، safety.
## O-09: Proposal/Risk/Mission contracts
**يُغلق بتحديد:** الحقول، invariants، expiration، version، وهل SL/TP يقترحه Engine أم تشتقه Risk.
**تعتمد عليه:** Decision، Risk، Execution.
## O-10: Risk policy baseline
**يُغلق بتحديد:** كل limits والصيغ وترتيب checks وسلوك missing account data.
**تعتمد عليه:** sizing، approvals، portfolio safety.
## O-11: Broker execution matrix
**يُغلق بتحديد:** netting/hedging support، filling modes، pending expiry، stops/freeze handling، slippage/retry، partial fills.
**تعتمد عليه:** IBroker وExecution وtests.
## O-12: OCO state machine
**يُغلق بتحديد:** states/events، atomicity assumptions، race handling، cancellation owner، recovery.
**تعتمد عليه:** OCO Engine، Execution، Position Management.
## O-13: Position Management policy
**يُغلق بتحديد:** allowed actions، owner، triggers، manual override، emergency scope.
**تعتمد عليه:** runtime، broker actions، audit.
## O-14: Runtime cadence وfreshness
**يُغلق بتحديد:** OnTick/OnTimer/new-bar responsibilities، maximum data age، H1/M15 alignment، market closed behavior.
**تعتمد عليه:** snapshot health، dedupe، latency.
## O-15: Recovery and reconciliation
**يُغلق بتحديد:** broker-as-source-of-truth rules، orphan detection، mission/order correlation، safe startup state.
**تعتمد عليه:** production readiness.
## O-16: Audit/replay standard
**يُغلق بتحديد:** event schema، sequencing، clock، versioning، persistence، integrity، replay acceptance criteria.
**تعتمد عليه:** compliance، debugging، live/backtest parity.

**بوابة البدء:** لا يلزم إغلاق كل تحسين مستقبلي، لكن O-01 إلى O-12 وO-14 وO-16 يجب أن تكون Accepted قبل تنفيذ pipeline الأساسية. O-13 وO-15 يجب إغلاقهما قبل أي تداول live.

* * *
# 5) مخاطر التنفيذ في MQL5
## 5.1 Immutability ليست مضمونة تلقائياً
MQL5 لا يقدم نموذج records immutable مريحاً. تمرير struct أو object قابل للتعديل قد يكسر ختم Context دون قصد، خصوصاً عند مشاركة references.

**الخطر:** Decision أو Dashboard تغير state مرجعية.
**المطلوب:** عقد read-only واضح، copy semantics معلنة، وعدم كشف mutable internals.
## 5.2 الواجهات وDependency Injection محدودة مقارنةً بلغات مؤسسية
يمكن بناء abstract classes/interfaces، لكن ownership والإنشاء والحقن اليدوي تصبح معقدة.

**الخطر:** singleton/global state واختلاط adapters بمنطق المجال.
**المطلوب:** composition root واحد في EA، وعقود صغيرة لـ IBroker وIAccountInfo وIRecordSink وIClock وIMarketData.
## 5.3 إدارة الذاكرة وobject ownership
الكائنات المنشأة ديناميكياً تحتاج delete واضحاً، والمصفوفات/النسخ قد تكون مكلفة أو ملتبسة.

**الخطر:** leaks أو dangling references أو state مشترك غير مقصود.
**المطلوب:** owner موثق لكل object، وتفضيل value types الصغيرة حين يكون ذلك آمناً.
## 5.4 Indicator handles وCopyBuffer
handles قد تفشل، البيانات قد لا تكون جاهزة، BarsCalculated قد يكون ناقصاً، والـ timeframe/symbol قد يتغير.

**الخطر:** snapshot تبدو صحية وهي مبنية على EMPTY\_VALUE أو bar غير مكتملة.
**المطلوب:** lifecycle صريح للـ handles، warmup، readiness، release، freshness، وفشل مغلق.
## 5.5 مزامنة H1 وM15
OnTick لا يضمن وصول بار جديد أو ترتيب تحديث كل series في اللحظة نفسها.

**الخطر:** H1 swing من دورة، وM15 monitor من دورة أخرى، أو استخدام forming bar في live واختلافه في backtest.
**المطلوب:** bar identities وclosed-bar policy وsnapshot coherence check.
## 5.6 \_Point ليس Tick Size دائماً
في بعض الذهب والمؤشرات والأسهم، SYMBOL\_TRADE\_TICK\_SIZE قد يختلف عن `_Point`. كما أن Digits لا تكفي لتطبيع سعر صالح للوسيط.

**الخطر:** invalid price/stops أو قياس مسافة خاطئ.
**المطلوب:** حساب مالي بـ tick size/tick value، وعرض point/pip كتحويل UI فقط.
## 5.7 Tick value والعملات
SYMBOL\_TRADE\_TICK\_VALUE قد يختلف للربح والخسارة أو يتأثر بعملة الحساب والرمز.

**الخطر:** sizing خاطئ.
**المطلوب:** استخدام خصائص الرمز والتحقق أو OrderCalcProfit/OrderCalcMargin ضمن adapter قابل للاختبار.
## 5.8 Stops level وFreeze level ديناميكيان
قد يرفض الوسيط تعديلاً أو pending order رغم أن الحساب النظري صحيح، وقد تتغير القيود.

**الخطر:** mission معتمدة غير قابلة للتنفيذ.
**المطلوب:** preflight قريب من الإرسال، دون السماح لـ Execution بتوسيع risk بصمت. إذا لزم تعديل جوهري تُرفض المهمة أو تعاد إلى Risk.
## 5.9 Volume min/max/step
NormalizeDouble لا يكفي لتطبيع الحجم.

**الخطر:** upsizing غير مقصود عند rounding أو invalid volume.
**المطلوب:** rounding policy محافظة (عادةً down ضمن الحد) مع إعادة حساب risk بعد التطبيع.
## 5.10 Hedging مقابل Netting
في netting، أمر جديد قد يدمج أو يعكس position؛ في hedging توجد tickets متعددة.

**الخطر:** OCO وpartial close وCLOSE ALL تعمل بصورة مختلفة جذرياً.
**المطلوب:** AccountMode capability gate، أو إعلان أن v1 تدعم نمطاً واحداً فقط.
## 5.11 OCO غير ذري على معظم الوسطاء
بين fill طرف وإلغاء الآخر توجد نافذة race، وقد يمتلئ الطرفان في سوق سريع.

**الخطر:** exposure مزدوج.
**المطلوب:** state machine وOnTradeTransaction reconciliation وخطة emergency compensation مع audit.
## 5.12 Event model وإعادة الدخول المنطقي
OnTick وOnTimer وOnTradeTransaction وOnChartEvent قد تحدث بينها تفاعلات وتحديثات state معقدة حتى لو كان التنفيذ thread-like محدوداً.

**الخطر:** duplicate order أو command مرتين.
**المطلوب:** idempotent command processing، mission IDs، ومخزن حالة واحد.
## 5.13 Strategy Tester ليس وسيطاً حقيقياً
retcodes، latency، slippage، partial fills، freeze behavior وجودة ticks لا تطابق live دائماً.

**الخطر:** backtest جيد وعقد تنفيذ غير مختبر.
**المطلوب:** broker simulator واختبارات adapter منفصلة، ثم demo soak test.
## 5.14 File I/O وAudit
الكتابة قد تفشل أو تتأخر، ومسارات الملفات وصلاحياتها تختلف في tester/VPS.

**الخطر:** فقدان audit أو إيقاف التداول بسبب sink بطيء.
**المطلوب:** IRecordSink بسياسة failure واضحة، buffering محدود، وتسلسل أحداث قابل للمصالحة.

* * *
# 6) خطة التنفيذ المقترحة، ترتيب القراءة = ترتيب البناء
## المرحلة 0: Architecture Baseline والعقود
**\[توضيح لازم قبل التنفيذ، وليس تغييراً للسلوك\]** تثبيت glossary، Decision Index، enums، schemas، invariants، units، IDs، clocks، errors، وdependency rules.

**Seams:** IClock، IMarketData، IAccountInfo، IRecordSink.
**Definition of Done:** كل عقد له owner، inputs/outputs، validity، failure behavior، version؛ وكل قرار مفتوح أساسي Accepted؛ ولا توجد قيم مالية بلا وحدة.
## المرحلة 1: Market Reading
بناء Readers مستقلة بالترتيب: market data readiness، Structure H1، Fibonacci H1، nearest levels، Trend/Regime، ثم Liquidity/Momentum/Volatility/Session/SMC حسب القرار النهائي.

**Seams:** IMarketData، IIndicatorProvider أو adapters أصغر، IClock، IRecordSink.
**Definition of Done:** fixtures ثابتة تنتج نفس facts؛ لا Reader يرسل أوامر أو يختار trade؛ handles تُدار بأمان؛ forming/closed bars محسومة؛ حالات insufficient/stale data تفشل مغلقة.
## المرحلة 2: TradingContext
بناء ContextBuilder باعتباره الكاتب الوحيد، ثم validation، coherence، health، seal، snapshot identity.

**Seams:** reader interfaces، IClock، IRecordSink.
**Definition of Done:** لا يمكن للطبقات اللاحقة تعديل snapshot عبر API؛ كل fact يحمل source/timeframe/bar identity؛ snapshot غير المتماسكة لا تُختم كHealthy؛ serialization round-trip ناجح.
## المرحلة 3: Decision Authority والمحركات
بناء NoTrade أولاً، ثم regime/policy matrix، ثم Trend Engine، ثم OCO Breakout/Reversal policy، ثم Manual intent routing.

**Seams:** IDecisionEngine، IPolicyProvider، IRecordSink.
**Definition of Done:** snapshot واحدة تنتج نتيجة حتمية واحدة؛ لا يعمل أكثر من Engine؛ كل rejection/no-trade له reason code؛ manual لا يتجاوز gates؛ tests تغطي التعارضات والحدود thresholds.
## المرحلة 4: Risk / Sizing / Portfolio Gates
بناء validation، SL/TP contract، cost-aware RR، sizing، volume rounding، account/exposure limits، ثم ApprovedMission immutable.

**Seams:** IAccountInfo، ISymbolInfo، ICostModel، IPortfolioState، IRecordSink.
**Definition of Done:** لا upsizing بعد normalization؛ كل approval قابل لإعادة الحساب من audit inputs؛ missing/invalid account data يرفض؛ tests تشمل FX/Gold/Index و3/5 digits وtick-size anomalies.
## المرحلة 5: Execution / Broker Integration
ابدأ بـ fake broker، ثم preflight، normalization، request mapping، retcode handling، idempotency، pending orders، ثم OCO state machine.

**Seams:** IBroker، IAccountInfo، ISymbolInfo، IClock، IRecordSink.
**Definition of Done:** لا request بلا ApprovedMission صالحة؛ retries لا تكرر exposure؛ كل retcode مصنف؛ stop/freeze/volume checks تعمل؛ OCO races مختبرة؛ netting/hedging scope enforced.
## المرحلة 6: Position Management
بناء state reconciliation، policy actions، modify/partial/close، emergency flat، ثم ownership transition من execution إلى management.

**Seams:** IBroker، IPositionRepository، IClock، IRecordSink.
**Definition of Done:** كل position مرتبطة بمهمة أو مصنفة orphan؛ كل action idempotent ومدقق؛ scope لـ CLOSE ALL محسوم؛ restart لا يكرر action.
## المرحلة 7: Operations / Runtime / UI
بناء orchestrator، cadence، warmup، dedupe، health، recovery، ثم dashboard وmanual command gateway وTelegram وفق scope.

**Seams:** IClock، IControlCommandSource، IHealthSink، IRecordSink، وواجهات read-only للطبقات.
**Definition of Done:** UI لا تكتب domain state مباشرة؛ الأزرار تنتج commands بهوية وصلاحية وaudit؛ dashboard تعرض source timeframe/freshness/reason؛ safe startup وhalt/recovery مختبرة.
## المرحلة 8: Audit / Persistence / Replay
واجهة IRecordSink يجب أن توجد منذ المرحلة 0، لكن implementation الكاملة وReplay تُغلق هنا بعد استقرار event schemas.

**Seams:** IRecordSink، IEventStore، IReplayClock، IBrokerSimulator.
**Definition of Done:** event chain كاملة لكل cycle؛ live/backtest يستخدمان schemas نفسها؛ replay يعيد Decision/Risk deterministically ضمن tolerance معلنة؛ schema versioning وcorrelation IDs واختبارات migration موجودة.
## بوابات الإطلاق
*   **Unit/Contract Gate:** كل طبقة تجتاز fixtures وحدودها دون MT5 terminal حيث يمكن.
*   **Strategy Tester Gate:** اختبارات تاريخية deterministic مع datasets محفوظة.
*   **Demo Gate:** soak test، restart/recovery، disconnect، widened spread، rejects، OCO race.
*   **Live Gate:** لا يُفتح إلا بعد موافقة صريحة، limits محافظة، kill switch، وaudit verified.

* * *
# Proposals، لا تُطبق دون اعتماد
## التصميم الموحد المقترح للرؤية الجديدة
### 1\. Market Intelligence
**\[توضيح للمعمارية\]** القرّاء ينتجون حقائق فقط: Trend، Regime، Structure/Location، Fibonacci، nearest support/resistance، Momentum، Liquidity، Volatility، Session وSMC. H1 يملك البنية والمستويات، وM15 يملك المراقبة التشغيلية.

**\[تغيير معماري مقترح P-09\]** المسافة ليست “محسوبة على timeframe”. المستوى يحمل أصل H1، أما المسافة فتقاس من مرجع سعري معلن وقت snapshot. المقترح: استخدام executable quote للقرار والتنفيذ، مع عرض M15 closed-bar distance كسياق على الداشبورد. يمنع ذلك الخلط الحالي بين مصدر المستوى ومصدر سعر القياس.
### 2\. TradingContext
**\[توضيح للمعمارية\]** ContextBuilder هو الكاتب الوحيد. يختم snapshot متماسكة تحمل symbol، bar identities لـ H1/M15، الأسعار، freshness، health، وحدات القياس، نسخة الخوارزميات ونتائج جميع القرّاء. أي نقص أو stale/incoherent data ينتج Wait.
### 3\. Market Classifier وPolicy
**\[تغيير معماري مقترح P-10\]** فصل تصنيف السوق عن اختيار المحرك:

1. Classifier ينتج حالة واحدة: Trend/Breakout/Reversal/Wait مع evidence وconfidence.
2. Policy تطبق precedence matrix وحالات التعارض.
3. Router يختار محركاً واحداً فقط.

القاعدة المحافظة المقترحة عند تساوي أو تعارض الحالات هي Wait، لا اختيار أعلى score بصورة ضمنية. يجب تثبيت thresholds وhysteresis قبل التنفيذ.
### 4\. Authority Gate: AUTO مقابل MANUAL
**\[توضيح للمعمارية\]** الوضع عالمي وواضح لكل symbol/instance:
*   `AUTO_ACTIVE`: يسمح للنظام بتحويل فرصة صالحة إلى intent آلية.
*   `MANUAL_OBSERVE`: يحلل ويعرض فقط؛ الحالة التنفيذية `Awaiting Human Decision`.
*   `MANUAL_ARMED`: يوجد طلب بشري one-shot صالح بمهلة محددة.
*   `PAUSED`: قراءة وعرض بلا إنشاء مهام.
*   `HALTED`: منع الأوامر الجديدة، والسماح فقط بإجراءات السلامة المحددة.

**\[تغيير معماري مقترح P-11\]** لا تستخدم boolean واحداً لـ AUTO/MANUAL. استخدم state machine تمنع الانتقال المبهم، وتلغي أي Manual Request عند انتهاء المهلة أو تغير الرمز أو تغير الحالة السوقية أو restart ما لم تتم المصالحة.
### 5\. الفرصة وطريقة الدخول والمحركات
**\[توضيح للمعمارية\]** قرار السوق ينتج Opportunity واحدة: Trend أو Breakout أو Reversal أو Wait. بعدها يختار Entry Planner طريقة الدخول Market أو Pending وفق قواعد الفرصة والوضع AUTO/MANUAL.
*   `Market`: دخول اتجاهي واحد بعد الاعتماد.
*   `Pending`: ينتقل إلى Pending Entry Builder لتطبيق Two Side أو One Side.

Trend Engine يختص بمنطق فرصة Trend. ومحرك OCO/Pending يختص ببناء الدخول المعلق لأي فرصة اختير لها `Entry Method = Pending`. اسم OCO لا يصبح بديلاً عن نوع الفرصة.

أمثلة التكوين الرسمية:
*   `Breakout → Pending → Two Side`: غالباً Buy Stop وSell Stop، مع OCO cancellation.
*   `Breakout → Pending → One Side → Buy Only`: Buy Stop واحد.
*   `Breakout → Pending → One Side → Sell Only`: Sell Stop واحد.
*   `Reversal → Pending → One Side → Buy Only`: Buy Limit واحد.
*   `Reversal → Pending → One Side → Sell Only`: Sell Limit واحد.
*   `Reversal → Pending → Two Side`: زوج Limit مع OCO cancellation، بشرط تعريف هندسة الأسعار وصلاحية الرجلين وقواعد السباق.
*   `Trend → Market`: دخول سوق اتجاهي.
*   `Trend → Pending → One Side`: دخول اتجاهي معلق عند سعر يحدده العقد.

**\[توضيح للمعمارية\]** `Two Side` و`One Side` هما Pending Entry Policy. أما `Buy Only` و`Sell Only` فهما اتجاه One Side. Proposal Validator ملزم بالتحقق من توافق Opportunity وEntry Method وPending Policy والاتجاه ونوع الأمر والأسعار مع quote وقيود الوسيط.

المحرك لا يحسب اللوت النهائي، لا يرسل أمراً، ولا يدير الصفقة بعد fill. Scalping هو Execution/Risk Profile داخل OCO Manual Request، وليس Engine ولا market state.
### 6\. العقود في AUTO
**\[توضيح للمعمارية\]** النظام ينتج `AutoTradeIntent`، ثم Risk Authority تشتق lot وSL وTP، تتحقق من RR والتكاليف والتعرض، وتنتج `ApprovedMission` أو `Rejected`.
### 7\. العقود في MANUAL
**\[تغيير معماري مقترح P-12\]** الطلب البشري ينتج `ManualTradeIntent` منفصلاً يحتوي: engine، policy، side mode، fixed lot، entry specification، SL، TP، expiry، actor، snapshotId وrequestId.

القواعد المقترحة:
*   fixed lot وSL وTP لا يعاد تحسينها آلياً.
*   لا rounding إلى أعلى للحجم.
*   أي تطبيع يغير المعنى المالي خارج tolerance معلنة يؤدي إلى رفض، لا تعديل صامت.
*   Hard Safety تتحقق من volume min/max/step، margin، executable prices، stops/freeze، spread/slippage cap، account mode والحد الكارثي.
*   تغير snapshot لا يبدل الطلب تلقائياً؛ إما يبقى صالحاً ضمن expiry والانحراف المسموح أو يرفض.
### 8\. Execution وOCO
**\[تغيير معماري مقترح P-13\]** Execution Adapter يرسل أوامر broker فقط، بينما `OCO Coordinator` يملك lifecycle: staging، إرسال الطرفين، acknowledgement، partial/fill، إلغاء الطرف المقابل، race recovery وterminal state. هذا يمنع OCO Engine من أن يصبح كاتباً ثانياً لحالة الوسيط.

`Single-Side Pending` يمر عبر المحرك وpipeline نفسيهما، لكنه لا يدخل OCO cancellation state machine لعدم وجود طرف مقابل. لا تعديل آلي لـ SL/TP أو توسيع risk عند رفض الوسيط؛ تعاد النتيجة Rejected أو Requires New Approval.
### 9\. Position Management
**\[توضيح للمعمارية\]** تبدأ بعد fill وتملك سياسة الصفقة حتى الإغلاق: initial protection verification، hold، trailing stop، breakeven، partial close، close وemergency flat.

**\[تغيير معماري مقترح P-14\]** كل Mission تحمل `ManagementPolicyId` ثابتاً عند الاعتماد. في AUTO يختاره النظام، وفي MANUAL يختاره المتداول أو يستخدم profile معلناً. لا يجوز لـ Position Manager اختراع trailing policy بعد الدخول.
### 10\. Operations وDashboard وTelegram
**\[توضيح للمعمارية\]** الداشبورد projection فقط ويعرض:
*   الحالة: Trend/Breakout/Reversal/Wait.
*   AUTO/MANUAL وAwaiting Human Decision/Armed/Paused/Halted.
*   المحرك والمسار المختاران.
*   H1 structure وM15 monitor مع bar time/freshness.
*   nearest support/resistance والمسافة ووحدة القياس.
*   momentum/liquidity/volatility/confidence.
*   سبب المنع أو الرفض.
*   risk/account/exposure.
*   mission، orders، positions وmanagement state.

**\[تغيير معماري مقترح P-15\]** Telegram outbound-only في v1: Market State Change، Manual Opportunity، Mission Approved/Rejected، Order/Fill/OCO Event، Protection Change، Closeout، Halt وHealth Failure. استقبال أوامر Telegram يؤجل حتى وجود authorization وdedupe وaudit مناسبين.
### 11\. Audit الكامل
**\[توضيح للمعمارية\]** كل دورة تسجل correlation chain واحدة:

`MarketFacts → SnapshotSealed → Classification → AuthorityState → Intent/Wait → EngineProposal → RiskOrSafetyDecision → ApprovedMission/Rejected → BrokerRequest/Result → OCOEvent → PositionAction → Closeout`.

كل منع يسجل reason code وحقائق القرار، وكل تدخل بشري يسجل actor/requestId والقيم الأصلية والقيم normalized. اللوج ليس رسائل حرة فقط؛ الأحداث typed وقابلة للـ replay، مع نص مختصر للعرض البشري.
## حدود الملكية النهائية
*   Reader يكتب ReaderResult الخاص به فقط.
*   ContextBuilder يكتب TradingContext وصحتها فقط.
*   Classifier يكتب MarketClassification فقط.
*   Policy/Router يختار المسار والمحرك فقط.
*   AUTO Authority أو Manual Command Gateway يكتب TradeIntent، ولا يجتمعان لنفس الدورة.
*   Engine يكتب Proposal فقط.
*   Risk Authority تكتب القيم النهائية في AUTO.
*   Hard Safety Authority تقبل أو ترفض القيم البشرية في MANUAL دون تحسينها.
*   Execution Adapter يملك broker requests/results.
*   OCO Coordinator يملك pending-pair lifecycle.
*   Position Manager يملك policy actions بعد fill.
*   Dashboard وTelegram قرّاء للأحداث، وليسا مصدر حقيقة.
*   Audit Event Store هو سجل الرحلة، والوسيط هو مصدر حقيقة الأوامر والمراكز عند المصالحة.
## التعارضات التي أغلقها هذا التصور
*   المحركات اثنان فقط؛ Scalping profile وليس محركاً.
*   Breakout/Reversal حالتان داخل OCO Engine وليستا محركين متنافسين.
*   AUTO/MANUAL يحددان سلطة القرار، ولا يغيران عدد المحركات.
*   MANUAL يجمد التنفيذ الآلي للفرصة، لكنه لا يعطل Hard Safety.
*   Opportunity لا تختلط بطريقة الدخول: Trend/Breakout/Reversal تجيب لماذا، وMarket/Pending تجيب كيف.
*   داخل Pending فقط: Two Side يشغّل OCO الحقيقي، وOne Side يتطلب Buy Only أو Sell Only ولا يشغّل إلغاءً متبادلاً.
*   Risk داخل Engine تلغى؛ Engine قد يرفق risk hints فقط.
*   إدارة الصفقة لا تبدأ قبل fill، ولا تدير OCO pending lifecycle.
*   كل snapshot تنتج classification واحدة ومساراً واحداً أو Wait.
## قرارات لا تزال تحتاج اعتماد المالك
1. هل MANUAL fixed lot يسمح بتجاوز **سقف المخاطرة المعتاد** ضمن hard catastrophic cap مستقل، أم يجب رفضه عند سقف المخاطرة المعتاد أيضاً؟
2. هل fixed SL/TP أسعار مطلقة، مسافات points/ticks، أم يدعم النظام الشكلين مع نوع صريح؟
3. هل الدخول اليدوي Market أم Pending فقط، وما أنواع pending المسموحة لكل Trend/Breakout/Reversal؟
4. ما scope للوضع: لكل EA instance، لكل symbol، أم للحساب كله؟
5. ما expiry للفرصة والطلب اليدوي، وما tolerance لتغير السعر قبل الرفض؟
6. هل trailing/breakeven إلزاميان أو profile يختاره المتداول عند التسليح؟
7. ما الحدود الكارثية غير القابلة للتجاوز: margin level، max lot، max loss، daily loss، spread وslippage؟
8. هل v1 تدعم hedging فقط أم netting فقط أم كليهما؟
9. هل Telegram تنبيهات فقط في v1؟ المقترح نعم.
10. اعتماد مصدر سعر المسافة: executable quote للقرار وM15 close للعرض السياقي، مع بقاء المستويات من H1.

هذه القرارات هي الحد الأدنى لإخراج Data Contracts وState Machines وDecision Matrix من دون تخمين.
## P-01: فصل Proposal عن ApprovedMission
**\[تغيير معماري مقترح\]** اجعل TradeProposal immutable، وRisk تنتج ApprovedMission جديدة بدلاً من تعديل proposal.

**التبرير:** يحمي single writer ويجعل audit/replay أوضح.
**الأثر:** إضافة contract بين Risk وExecution، وتبسيط منع execution قبل الاعتماد.
## P-02: ContextBuilder ككاتب وحيد
**\[توضيح يتحول إلى تغيير إن كان Readers يكتبون Context حالياً\]** Readers تعيد ReaderResult، وContextBuilder وحده يجمع ويختم.

**التبرير:** يمنع partial state وترتيب الكتابة غير الحتمي.
**الأثر:** interfaces أوضح واختبارات أسهل.
## P-03: فصل OCO policy عن OCO lifecycle
**\[تغيير معماري مقترح\]** OCO Engine تقترح legs والسياسة؛ OCO Execution Coordinator يملك lifecycle والإلغاء والمصالحة.

**التبرير:** منطق السوق لا يجب أن يدير broker races.
**الأثر:** state machine إضافية، لكن حدود الملكية تصبح سليمة.
## P-04: استخدام PriceDistance typed لا أرقام خام
**\[تغيير معماري مقترح\]** تمثيل المسافة داخلياً بسعر/ticks مع تحويل صريح إلى points/pips للعرض.

**التبرير:** `_Point` ليس وحدة تداول آمنة لكل الأصول.
**الأثر:** تعديل contracts والحسابات والdashboard، وتحسن كبير في multi-asset safety.
## P-05: Health enum بأسباب، لا boolean
**\[تغيير معماري مقترح\]** حالات مثل Healthy / WarmingUp / Stale / Incoherent / MissingData / BrokerUnavailable / Halted.

**التبرير:** fail closed يحتاج سبباً قابلاً للتدقيق والعرض.
**الأثر:** Decision وOperations وDashboard وAudit.
## P-06: Control Command Gateway
**\[تغيير معماري مقترح\]** كل زر أو Telegram command يتحول إلى command typed مع actor/time/id/scope، ويمر عبر authorization وdedupe وaudit.

**التبرير:** يمنع أن تصبح UI كاتباً ثانياً للحالة.
**الأثر:** طبقة عمليات أنظف وتعافٍ أفضل بعد restart.
## P-07: حصر v1 في Account Mode واحد إن لزم
**\[تغيير نطاق مقترح\]** إن لم توجد حاجة فورية للنمطين، ابدأ بـ hedging أو netting واحد معلن بدلاً من دعم ناقص لكليهما.

**التبرير:** يقلل مخاطر OCO وposition lifecycle.
**الأثر:** capability gate في OnInit ووثيقة نطاق واضحة.

* * *
# قرار المراجعة النهائي
المعمارية الأساسية **واعدة وسليمة الاتجاه**، لكن العقود الحالية هي outline وليست implementation specification. لا أنصح بكتابة أي كود جديد قبل إغلاق القرارات O-01 إلى O-12 وO-14 وO-16، وإصدار النسخ الرسمية من Data Contracts وDecision Matrix وRisk Policy وOCO State Machine.

**التوقف المطلوب:** هذه الوثيقة هي Review Baseline فقط. لا تنفيذ، لا تعديل كود، ولا اعتماد لادعاءات التنفيذ الموجودة داخل `arc.md` حتى تُرفق ملفات المصدر والقرارات المرجعية وتصدر موافقة صريحة على المراجعة.