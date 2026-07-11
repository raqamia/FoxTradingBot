# FOX MQL5 | المرجع المعماري والبرمجي النهائي v1.0

# FOX MQL5 | المرجع المعماري والبرمجي النهائي v1.0
**الحالة:** Architecture Baseline جاهز للاعتماد، لا يبدأ التنفيذ قبل موافقة المالك الصريحة
**النطاق:** منصة تداول مؤسسية MQL5 للفوركس والذهب والمؤشرات
**المبدأ الحاكم:** Fail Closed، Single Writer، Deterministic Decisions، Full Auditability
> هذه الوثيقة هي المرجع الأعلى للمشروع. عند تعارض الكود أو مستند فرعي معها، تتوقف الميزة ويُفتح قرار معماري، ولا يُحل التعارض بافتراض برمجي.
* * *
# 1\. الرؤية والهدف
FOX نظام تداول متعدد الطبقات يقرأ السوق، يبني صورة موحدة ومختومة، يصنف الفرصة، يحدد طريقة الدخول، يمرر المهمة عبر المخاطر والسلامة، ينفذها عبر وسيط MQL5، ثم يديرها حتى الإغلاق مع سجل كامل قابل للمراجعة وإعادة التشغيل.

الأهداف الأساسية:

1. منع اختلاط قراءة السوق بقرار التداول أو التنفيذ.
2. إنتاج قرار واحد حتمي لكل Snapshot: Trend أو Breakout أو Reversal أو Wait.
3. دعم AUTO وMANUAL دون إنشاء مسارين برمجيين متناقضين.
4. دعم Market وPending، مع Two Side OCO حقيقي أو One Side.
5. حماية الحساب بقيود لا يمكن تجاوزها حتى في MANUAL.
6. توثيق الرحلة كاملة: سبب السماح، المنع، التنفيذ، الإدارة والإغلاق.
7. تحقيق parity تعاقدية بين Live وBacktest وReplay.
## 1.1 مبادئ غير قابلة للتفاوض
*   لا تنفيذ دون `ApprovedMission` صالحة وغير منتهية.
*   كل طبقة تكتب عقدها فقط، ولا تعدل مخرجات طبقة سابقة.
*   TradingContext مختومة بعد البناء.
*   Dashboard وTelegram projections وليسا مصدر حقيقة.
*   Broker هو مصدر حقيقة الأوامر والمراكز عند المصالحة.
*   أي بيانات ناقصة أو قديمة أو غير متماسكة تنتج Wait أو Rejection.
*   لا تعديل صامت للوت أو SL أو TP أو أسعار الدخول.
*   كل قرار أو تدخل بشري يحمل هوية وسبباً ووقتاً ونسخة استراتيجية.

* * *
# 2\. النموذج المفاهيمي الرسمي
## 2.1 لماذا، كيف، وكيف يُنشأ Pending

```plain
flowchart TD
    A[لماذا سندخل؟] --> B{Opportunity}
    B --> B1[Trend]
    B --> B2[Breakout]
    B --> B3[Reversal]
    B --> B4[Wait]
    B1 --> C{كيف سندخل؟}
    B2 --> C
    B3 --> C
    C --> C1[Market]
    C --> C2[Pending]
    C2 --> D{Pending Entry Policy}
    D --> D1[Two Side: OCO حقيقي]
    D --> D2[One Side]
    D2 --> D3[Buy Only]
    D2 --> D4[Sell Only]
```

العقود الثلاثة مستقلة:
*   `Opportunity`: سبب الصفقة، Trend أو Breakout أو Reversal أو Wait.
*   `EntryMethod`: Market أو Pending.
*   `PendingEntryPolicy`: TwoSide أو OneSide، وتُستخدم فقط عندما EntryMethod = Pending.
*   `OneSideDirection`: BuyOnly أو SellOnly، ومطلوبة فقط عند OneSide.

**OCO الحقيقي:** TwoSide فقط. يوجد أمران معلقان، وأول Fill مؤكد يطلق إلغاء الطرف الآخر. OneSide يمر عبر Pending pipeline نفسها، لكنه لا يدخل OCO cancellation lifecycle.
## 2.2 المحركات
يوجد محركان استراتيجيان في v1:

1. `Trend Engine`: يقيّم ويبني فرصة Trend.
2. `OCO Opportunity Engine`: يقيّم ويبني فرص Breakout وReversal.

بعد إنتاج Opportunity، توجد طبقة مستقلة `Entry Planner` تختار Market أو Pending. إذا اختارت Pending، يستخدم `Pending/OCO Builder` لبناء OneSide أو TwoSide. هذا الفصل يمنع خلط سبب الدخول بطريقة إنشاء الأوامر.

**اقتراح تسمية مؤسسية:** الاسم الأدق برمجياً للمحرك الثاني هو `Event Opportunity Engine`، مع إبقاء “OCO” اسماً تشغيلياً في الواجهة إن رغبت الإدارة. السبب: OCO وصف lifecycle لأمرين، وليس سبباً استراتيجياً للدخول.

* * *
# 3\. الرسم المعماري العام

```plain
flowchart TD
    MD[Market Data Adapters] --> MI[Market Intelligence Readers]
    MI --> CB[Context Builder]
    CB --> TC[Sealed TradingContext]
    TC --> CL[Opportunity Classifier]
    CL --> DP[Decision Policy and Conflict Resolver]
    DP --> AG{Authority Gate}
    AG -->|AUTO| AA[Auto Decision Authority]
    AG -->|MANUAL| MA[Manual Observe / Arm Gateway]
    AA --> SE[Strategy Engine]
    MA --> SE
    SE --> EP[Entry Planner: Market / Pending]
    EP -->|Market| MB[Market Entry Builder]
    EP -->|Pending| PB[Pending / OCO Builder]
    MB --> RG[Risk and Safety Gates]
    PB --> RG
    RG -->|Rejected| AU[Audit Event Store]
    RG -->|ApprovedMission| EX[Execution Adapter]
    EX --> OC[OCO Coordinator when TwoSide]
    EX --> PM[Position Manager after Fill]
    OC --> PM
    PM --> CO[Closeout and Performance Attribution]
    CO --> AU
    TC --> AU
    CL --> AU
    AG --> AU
    SE --> AU
    EP --> AU
    EX --> AU
    OC --> AU
    PM --> AU
    AU --> DB[Dashboard Projection]
    AU --> TG[Telegram Alerts]
    AU --> RP[Replay / Backtest Analysis]
    KS[Independent Kill Switch] --> RG
    KS --> EX
    KS --> PM
```

* * *
# 4\. الهيكل التنظيمي وحدود الملكية

```plain
flowchart LR
    O[Runtime Orchestrator] --> A[Market Intelligence]
    O --> B[Decision Domain]
    O --> C[Trade Domain]
    O --> D[Operations Domain]

    A --> A1[Readers]
    A --> A2[Context Builder]

    B --> B1[Classifier]
    B --> B2[Conflict Resolver]
    B --> B3[Trend Engine]
    B --> B4[OCO Opportunity Engine]
    B --> B5[Entry Planner]

    C --> C1[Risk Authority]
    C --> C2[Hard Safety Authority]
    C --> C3[Execution Adapter]
    C --> C4[OCO Coordinator]
    C --> C5[Position Manager]

    D --> D1[Audit and Persistence]
    D --> D2[Dashboard]
    D --> D3[Telegram]
    D --> D4[Recovery and Reconciliation]
    D --> D5[Shadow / Champion-Challenger]
```

## 4.1 Single Writer Matrix

| الحالة أو العقد | الكاتب الوحيد | ممنوع عليه |
| ---| ---| --- |
| ReaderResult | الـReader المختص | اختيار صفقة أو إرسال أمر |
| TradingContext + Health | ContextBuilder | تعديلها بعد الختم |
| OpportunityClassification | Classifier | اختيار اللوت أو التنفيذ |
| Conflict Resolution | Decision Policy | تغيير facts |
| TradeProposal | المحرك المختار | اعتماد المخاطر أو الإرسال |
| EntryPlan | Entry Planner/Builder | تعديل Opportunity |
| قيم AUTO النهائية | Risk Authority | الإرسال للوسيط |
| قبول MANUAL | Hard Safety Authority | تحسين رأي المتداول أو تغيير مدخلاته بصمت |
| BrokerRequest/Result | Execution Adapter | إعادة التصنيف أو sizing |
| OCO pending-pair state | OCO Coordinator | إدارة الصفقة بعد اكتمال fill |
| Position actions | Position Manager | إنشاء فرصة جديدة |
| Audit events | Event Store | اتخاذ قرار تداول |
| UI/Telegram | Projection/Notifier | كتابة domain state مباشرة |

* * *
# 5\. رحلة اتخاذ القرار

```plain
sequenceDiagram
    participant M as Market Data
    participant R as Readers
    participant C as ContextBuilder
    participant D as Decision Authority
    participant A as AUTO/MANUAL Authority
    participant E as Strategy Engine
    participant P as Entry Planner
    participant K as Risk/Safety
    participant X as Execution
    participant O as OCO Coordinator
    participant G as Position Manager
    participant U as Audit

    M->>R: H1/M15 data + live quote
    R->>C: Typed ReaderResults
    C->>C: coherence, freshness, health, seal
    C->>U: SnapshotSealed
    C->>D: TradingContext
    D->>D: Trend/Breakout/Reversal/Wait
    D->>U: Classification + evidence
    alt Wait or conflict
        D->>U: NoTrade reason
    else Opportunity exists
        D->>A: Opportunity
        alt AUTO
            A->>E: Authorized auto intent
        else MANUAL Observe
            A->>U: AwaitingHumanDecision
        else MANUAL Armed
            A->>E: One-shot manual intent
        end
        E->>P: TradeProposal
        P->>P: Market or Pending; One/Two Side
        P->>K: EntryPlan
        alt Rejected
            K->>U: Risk/Safety rejection
        else Approved
            K->>X: ApprovedMission
            X->>U: Broker requests/results
            opt TwoSide
                X->>O: Register paired orders
                O->>O: fill/cancel/race/recovery
                O->>U: OCO lifecycle events
            end
            X->>G: Filled position
            G->>G: initial SL, trailing, partial, close
            G->>U: Position actions + closeout
        end
    end
```

## 5.1 ترتيب القرار الحتمي
1. فحص جاهزية البيانات والـhandles.
2. بناء Market Facts.
3. فحص H1/M15 coherence وfreshness.
4. ختم TradingContext.
5. تصنيف Opportunity.
6. تطبيق thresholds وhysteresis وconflict policy.
7. إذا تعارضت فرص متقاربة أو نقص دليل حاسم: Wait.
8. تحديد Authority: AUTO أو MANUAL.
9. بناء TradeProposal من محرك واحد فقط.
10. اختيار Entry Method.
11. عند Pending: اختيار TwoSide أو OneSide واتجاه OneSide.
12. احتساب Expected Edge بعد التكاليف.
13. Risk/Safety decision.
14. ApprovedMission أو Rejection.
15. التنفيذ والمصالحة.
16. إدارة الصفقة حتى Closeout.
17. Attribution وتسجيل النتيجة مقابل سبب الدخول ونسخة الاستراتيجية.

* * *
# 6\. Market Intelligence وTradingContext
## 6.1 توزيع الأطر الزمنية
*   H1: Structure، confirmed swings، Fibonacci anchors، support/resistance universe، structural regime.
*   M15: operational monitoring، momentum، liquidity reaction، candle confirmation، proximity behavior.
*   Live quote: executable distance والـpreflight.
*   H4: ليس default في هذا workflow، ولا يدخل دون قرار معماري جديد.
## 6.2 المسافات والوحدات
المستوى يحمل مصدره `H1`. المسافة لا “تنتمي” إلى timeframe، بل تُحسب من مرجع سعر معلن:
*   للقرار والتنفيذ: executable quote المناسب، Ask للشراء وBid للبيع.
*   للسياق البصري: يمكن عرض المسافة من آخر M15 closed bar مع تسمية واضحة.
*   داخلياً: price/ticks باستخدام `SYMBOL_TRADE_TICK_SIZE`.
*   points/pips: عرض فقط، بتحويل صريح حسب الرمز.
## 6.3 صحة Snapshot
الحالات الرسمية:

`Healthy / WarmingUp / MissingData / Stale / Incoherent / InvalidIndicator / BrokerUnavailable / Halted`.

لا يسمح بالقرار إلا عند Healthy. يجب أن تحمل Snapshot: symbol، snapshotId، server time، H1/M15 bar IDs، source timestamps، algorithm versions، build version، quote، spread، freshness، units وreason codes.

* * *
# 7\. AUTO وMANUAL
## 7.1 State Machine
`AUTO_ACTIVE / MANUAL_OBSERVE / MANUAL_ARMED / PAUSED / HALTED`.
*   AUTO\_ACTIVE: النظام يقرر lot وSL وTP وطريقة الدخول بعد جميع البوابات.
*   MANUAL\_OBSERVE: النظام يحلل ويعرض الفرصة، ولا ينفذ.
*   MANUAL\_ARMED: طلب بشري one-shot مرتبط بـsnapshot وexpiry.
*   PAUSED: قراءة وعرض فقط، بلا مهام جديدة.
*   HALTED: منع الدخول الجديد، والسماح بإجراءات الحماية والإغلاق المحددة.
## 7.2 Manual Intent
يحمل على الأقل: requestId، actor، symbol، opportunity، entry method، pending policy، direction، fixed lot، entry prices، SL، TP، management profile، snapshotId، createdAt، expiresAt، max price deviation.

في MANUAL:
*   لا يعاد تحسين fixed lot أو SL أو TP.
*   لا rounding إلى أعلى.
*   التطبيع الذي يغير المعنى خارج tolerance يؤدي إلى رفض.
*   Hard Safety تبقى إلزامية: volume step، margin، stops/freeze، spread، slippage، account mode، max catastrophic exposure.
*   تغير السعر أو انتهاء TTL أو تغير Opportunity يبطل الطلب وفق policy معلنة.

* * *
# 8\. Risk وHard Safety
## 8.1 AUTO Risk Authority
تشتق اللوت وSL وTP، وتحسب RR بعد spread والعمولة والانزلاق المتوقع، وتفحص exposure والارتباط وتركيز العملة وحدود اليوم وعدد الصفقات والهامش.
## 8.2 MANUAL Hard Safety
تتحقق ولا تحسن. النتيجة إما Accepted كما طُلب ضمن tolerance، أو Rejected بسبب typed.
## 8.3 قواعد أساسية
*   لا upsizing بعد volume normalization.
*   أي تغيير جوهري تطلبه قيود الوسيط يعيد المهمة للاعتماد.
*   لا تُعتبر RR صالحة قبل إدخال تكاليف التنفيذ.
*   Portfolio gate يرى المخاطر المجمعة لكل الرمز والعملات المترابطة.
*   Kill Switch مستقل عن Decision Authority.

* * *
# 9\. Execution وOCO وPosition Management
## 9.1 Execution
يقبل ApprovedMission فقط، ويتحقق فور الإرسال من tick size، volume step، stops level، freeze level، filling mode، expiration، market state، margin وaccount mode. كل request تحمل missionId وidempotency key وMagic Number.
## 9.2 OCO Coordinator
يعمل فقط مع Pending + TwoSide. الحالات الدنيا:

`Draft / Staging / LegAPlaced / BothPlaced / OneFilled / CancelRequested / Cancelled / DoubleFill / Recovered / Failed / Closed`.

أول fill مؤكد يطلق إلغاء الطرف الآخر. لأن OCO غير ذري لدى أغلب الوسطاء، يجب التعامل مع double fill، partial fill، failure to cancel، restart وdisconnect عبر `OnTradeTransaction` ومصالحة دورية. أي exposure مزدوج يدخل emergency policy ويسجل كاملاً.
## 9.3 Position Management
تبدأ الملكية بعد fill. كل Mission تحمل `ManagementPolicyId` ثابتاً يحدد initial protection، trailing، breakeven، partial close، time stop، close وemergency flat. لا يخترع المدير سياسة جديدة بعد الدخول.

* * *
# 10\. التحسينات المؤسسية والتداولية المعتمدة في التصميم
## 10.1 Strategy Versioning
كل Snapshot وProposal وMission وTrade Result تحمل نسخ Reader algorithms وDecision Policy وRisk Policy وManagement Policy. لا تحليل أداء بلا version attribution.
## 10.2 Shadow Mode
تشغيل AUTO كاملاً دون إرسال أوامر، مع تسجيل القرار والسعر الافتراضي والنتيجة. يستخدم قبل تفعيل استراتيجية جديدة وفي مراقبة الانحراف.
## 10.3 Champion / Challenger
Champion وحده مخول بالتنفيذ. Challenger يعمل Shadow على المدخلات نفسها. الترقية لا تتم من dashboard مباشرة، بل عبر قرار إصدار واختبارات قبول.
## 10.4 Expected Edge
كل Opportunity يجب أن تحمل:
*   expected edge بعد التكاليف.
*   invalidation condition.
*   time-to-live.
*   evidence and confidence.
*   adverse excursion budget.

لا تدخل فرصة بلا TTL أو invalidation.
## 10.5 Hysteresis وRegime Persistence
تستخدم thresholds منفصلة للدخول والخروج من الحالة، مع minimum persistence، لمنع التقلب بين Trend وBreakout وReversal كل شمعة.
## 10.6 Confidence Calibration
Confidence ليست زينة ولا score normalized فقط. تُعاير حسب symbol وsession وregime وstrategy version، وتُراقب reliability الفعلية دورياً.
## 10.7 Execution-Aware Edge
القرار يدخل spread، commission، expected slippage، stop distance، liquidity، news risk وlatency budget. إذا التهمت التكاليف edge، النتيجة Wait.
## 10.8 News وLiquidity Gates
بوابة أحداث وأوقات سيولة منخفضة قابلة للتهيئة. نقص بيانات الأخبار لا يسمح بافتراض أن السوق آمن إذا كانت البوابة إلزامية.
## 10.9 No-Trade Budget
حدود cooldown، max attempts per opportunity، max trades per session، ومنع إعادة الدخول إلى الفكرة نفسها دون invalidation ثم reset. الهدف منع churn والإفراط في التداول.
## 10.10 Live vs Backtest Drift
مراقبة فروق spread، slippage، fill ratio، rejection rate، latency، confidence calibration وperformance attribution. تجاوز الحدود ينقل النظام إلى Shadow أو Halt وفق severity.

* * *
# 11\. Operations وDashboard وTelegram
## 11.1 Dashboard
تعرض فقط من projections موثوقة:
*   Health وMode وKill Switch.
*   Opportunity وevidence وconfidence وTTL.
*   Entry Method وPending Policy والاتجاه.
*   H1 structure وM15 monitor وfreshness.
*   nearest support/resistance والمسافات ووحداتها.
*   spread، volatility، momentum، liquidity، session وnews gate.
*   Risk/Hard Safety state وسبب المنع.
*   Mission، orders، OCO lifecycle، positions وmanagement policy.
*   strategy versions وChampion/Challenger وShadow state.

الأزرار تنتج Typed Control Commands مع actor/time/id/scope، ولا تعدل domain state مباشرة.
## 11.2 Telegram
في v1: outbound-only. التنبيهات الأساسية: Market State Change، Manual Opportunity، expiry، Approval/Rejection، Order/Fill/OCO، protection change، closeout، drift، health failure وhalt. استقبال أوامر يؤجل حتى وجود authorization وdedupe وaudit قويين.

* * *
# 12\. Audit وReplay
السلسلة الرسمية:

`MarketFacts → SnapshotSealed → Classification → ConflictResolution → AuthorityState → Intent/Wait → EngineProposal → EntryPlan → Risk/SafetyDecision → ApprovedMission/Rejected → BrokerRequest/Result → OCOEvent → PositionAction → Closeout → PerformanceAttribution`.

كل Event typed وتحمل eventId، correlationId، causationId، sequence، timestamp، symbol، versions وreason code. اللوج النصي عرض بشري مشتق، وليس السجل القانوني الوحيد.

Replay يستخدم clock ومصادر بيانات وbroker simulator قابلة للحقن، ولا يصل إلى broker حقيقي. معيار القبول: نفس inputs والإصدارات تنتج نفس classification وproposal وrisk result ضمن tolerance معلنة للحسابات العشرية.

* * *
# 13\. قيود MQL5 المطلوبة في التصميم
*   Composition Root واحد في EA، بلا globals كسلطة قرار.
*   Interfaces صغيرة: `IMarketData`, `IIndicatorProvider`, `IClock`, `IAccountInfo`, `ISymbolInfo`, `IBroker`, `IRecordSink`, `IEventStore`.
*   ownership واضح لكل object وindicator handle، مع release مضمون.
*   فحص `BarsCalculated`, `CopyBuffer`, `CopyRates`, `EMPTY_VALUE` وwarmup.
*   استخدام tick size/tick value و`OrderCalcProfit/OrderCalcMargin`، لا `_Point` وDigits وحدهما.
*   فصل hedging عن netting بCapability Gate واختبارات مستقلة.
*   استخدام `OnTradeTransaction` للمصالحة، وعدم الاعتماد على نتيجة send وحدها.
*   idempotency لكل Mission وCommand لمنع duplicate orders.
*   persistence ذري قدر الإمكان، مع failure policy واضحة لـIRecordSink.
*   Strategy Tester ليس دليلاً كافياً على broker behavior؛ يلزم Demo soak test.

* * *
# 14\. ترتيب البناء حسب الأولوية
## P0: Architecture Contracts
تثبيت enums، schemas، units، IDs، reason codes، state machines، dependency rules وDecision Index.

**DoD:** لا حقل مالي بلا وحدة، ولا عقد بلا owner/validity/version/failure behavior.
## P1: Market Intelligence
Market data readiness، Structure H1، Fibonacci H1، nearest levels، Trend/Regime، Momentum، Liquidity، Volatility، Session، SMC.

**DoD:** نتائج حتمية من fixtures، handles آمنة، closed/forming policy محسومة، fail closed.
## P2: TradingContext
ContextBuilder، coherence، freshness، health، seal، serialization.

**DoD:** snapshot غير قابلة للتعديل عبر API، وكل fact يحمل source وbar identity.
## P3: Decision Domain
Classifier، conflict resolver، hysteresis، engines، Entry Planner، Wait-first tests.

**DoD:** نتيجة واحدة ومحرك واحد، وكل NoTrade له reason code، ولا يوجد multi-engine execution.
## P4: Risk/Safety
AUTO sizing، MANUAL validation، costs، exposure، normalization، ApprovedMission.

**DoD:** لا upsizing، وكل approval قابل لإعادة الحساب من audit inputs.
## P5: Execution/OCO
Fake Broker أولاً، preflight، request mapping، retcodes، retries، idempotency، Pending وOCO Coordinator.

**DoD:** لا request بلا mission، واختبارات race/double fill/restart/netting/hedging ناجحة ضمن النطاق.
## P6: Position Management
Reconciliation، initial protection، trailing، partial، close، emergency.

**DoD:** كل position مرتبطة بمهمة أو مصنفة orphan، وكل action idempotent ومدقق.
## P7: Operations
Runtime cadence، recovery، dashboard، commands، Telegram، kill switch، Shadow وdrift monitors.

**DoD:** UI لا تكتب state، وstartup/restart/disconnect/halt مختبرة.
## P8: Audit/Replay وإطلاق تجريبي
Event Store، replay، attribution، schema migration، tester datasets وDemo soak.

**DoD:** chain كاملة، deterministic replay، وlive/backtest schema parity.

* * *
# 15\. بوابات الاختبار والإطلاق
1. Contract Gate: schemas وinvariants وnegative tests.
2. Unit Gate: domain tests خارج MT5 حيث يمكن.
3. Integration Gate: fake broker وindicator adapters.
4. Strategy Tester Gate: datasets محفوظة ونتائج قابلة للتكرار.
5. Shadow Gate: قرارات live بلا تنفيذ.
6. Demo Gate: soak، restart، disconnect، spread widening، rejects، OCO races.
7. Limited Live Gate: limits محافظة ورمز واحد ونسخة Champion واحدة.
8. Production Gate: audit verified، drift monitoring، kill switch واختبارات recovery.

* * *
# 16\. قرارات الإعداد المطلوب تثبيتها قبل الكود
هذه ليست فراغات معمارية، بل قيم Policy يجب أن يعتمدها المالك:

1. Swing H1 algorithm وdepth وconfirmation وminimum range.
2. Universe الدعم والمقاومة وترتيب المناطق المتداخلة.
3. Regime thresholds وhysteresis وminimum persistence.
4. Entry Method matrix لكل Opportunity في AUTO.
5. هندسة الأسعار لكل Pending configuration.
6. Risk limits وcatastrophic limits وcorrelation policy.
7. Manual TTL وprice-deviation tolerance وصيغ SL/TP.
8. Management profiles وtrailing/breakeven/time stop.
9. Hedging/Netting scope في v1.
10. News provider/failure policy وTelegram scope.
11. Audit retention وintegrity standard.
12. Drift thresholds التي تنقل النظام إلى Shadow أو Halt.

كل قيمة تسجل في Decision Index بحالة `Proposed / Accepted / Superseded`، مع التاريخ والمالك والأثر والنسخة.

* * *
# 17\. قرار الاعتماد
هذه الوثيقة تؤسس بنية قابلة للتنفيذ دون تعارضات مفاهيمية: Opportunity منفصلة عن Entry Method، وPending Policy منفصلة عن اتجاه OneSide، وAUTO/MANUAL منفصلان عن المحركات، وRisk Optimization منفصلة عن Hard Safety، وOCO lifecycle منفصلة عن Position Management.

**لا يبدأ الكود قبل:** اعتماد هذه الوثيقة، تثبيت سياسات القسم 16، ثم إصدار Data Contracts وDecision Matrix وRisk Policy وOCO State Machine كملحقات معيارية. بعد ذلك يكون ترتيب القراءة هو نفسه ترتيب البناء الوارد في القسم 14.