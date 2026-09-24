Dart 生态的治理三件套：dart analyze（官方分析器）+ custom_lint（自定义规则）+ dart pub outdated（依赖治理），再用 DCM（原 Dart Code Metrics） 补齐复杂度/重复度度量，整个观测层就能外包出去。Dart 的独特优势是 analysis_options.yaml 是官方标准——所有 lint 规则、架构边界、语言规范都可以写成一份 YAML，CI 和 IDE 共用同一套事实源，这比 Rust/Java 的”规则散落在多工具里“要清爽得多。
Dart 治理工具全景表
层级	解决什么问题	推荐工具	接入成本	开源状态	备注
官方分析器	代码风格、惯用法、错误检测	dart analyze + dart format	极低，Dart SDK 内置	官方内置	第一道闸门，配合 analysis_options.yaml 生效
严格规则集	开箱即用的高质量规则集	flutter_lints / verygoodanalysis	低，pubspec 一行	✅ 开源	verygoodanalysis 由 Very Good Ventures 维护，规则比官方更严
自定义 Lint	团队架构约定（禁止跨层 import 等）	custom_lint	中，需写 Dart 代码	✅ 开源（Invertase 出品）	把”不许这么写“变成可执行 lint，支持 quick fix
架构边界守护	分层架构、模块依赖规则	custom_lint + 自定义规则	中	✅ 开源	用 custom_lint 实现 ArchUnit 式的架构测试
深度分析插件	写 analyzer 插件做更复杂检查	analyzer_plugin	高	✅ 开源	VGV 有教程，适合 custom_lint 不够用的场景
复杂度/度量	圈复杂度、嵌套深度、重复代码、死代码	DCM（原 dartcodemetrics）	低	部分开源，高级规则收费	补齐官方 linter 覆盖不到的度量维度
依赖清理	找出无用/过时的依赖	dart pub outdated + dep_audit	极低	✅ 官方内置	dart pub outdated 官方推荐使用
依赖图可视化	看清”谁依赖谁“	dart pub deps	极低	✅ 官方内置	dart pub deps —style=compact 输出依赖树，可转 Graphviz
包质量评分	对外陈述”这个包健康“	pana	低	✅ 官方出品	本地跑分和 pub.dev 一致，是包治理的硬指标
License 合规	依赖的 License 是否可用	Very Good CLI license checker	低	✅ 开源	本地检测，不联网，基于 Dart 分析器
Monorepo 治理	多 package 统一脚本/版本/发布	Melos（Invertase） + Pub Workspaces	中	✅ 开源	Melos 负责脚本/版本/发布，Pub Workspaces 是 Dart 3.6+ 原生能力
代码知识图谱	类图、模块图生成	dcdg（Dart Class Diagram Generator）	中	✅ 开源	输出 PlantUML 类图，配合 Mermaid 也能用
三个核心工具的深度说明
① analysis_options.yaml + custom_lint —— Dart 的”ArchUnit“
这是 Dart 治理里最有价值的一块。官方 analysis_options.yaml 是所有 lint/架构规则的单一事实源，IDE 和 CI 完全一致。在此基础上用 custom_lint 可以写出诸如”UI 层不得 import data/repository“、”所有 public class 必须有文档注释“、”禁止使用 print，统一走 log“这类团队约定，并且支持 quick fix 让开发者一键修复。相比 Rust 的 Dylint、Java 的 ArchUnit，custom_lint 的优势是规则直接跑在 analyzer 上，错误位置和 quick fix 的精度都是编译器级。
② DCM（Dart Code Metrics）—— 补齐”度量“短板
官方 linter 擅长”风格/错误“，但对”复杂度、重复度、死代码“这类度量维度覆盖较弱。DCM 提供 400+ 规则，包括圈复杂度、函数长度、嵌套深度、重复代码块、未使用代码等，是目前 Dart 生态最完整的度量工具。注意它已转为”核心开源 + 高级功能付费“模式，基础度量对小团队依然免费够用。
③ Melos + Pub Workspaces —— Dart 的 Monorepo 双方案
Monorepo 治理在 Dart 里现在是”官方 + 社区“双轨：
• Pub Workspaces（Dart 3.6+ 原生）：解决依赖统一、跨包引用、共享 analysis_options.yaml，官方方案首选
• Melos（Invertase 出品）：解决跨包脚本执行、版本号统一管理、changelog 生成、发布流水线，功能更完整
大型 Flutter 项目（如 FlutterFire）生产环境用的就是 Melos，成熟度足够。
第二大脑载体：Backstage 还是 pub.dev + Melos
Dart 生态有个天然优势——pub.dev 的包评分体系本身就是一套”可对外陈述的质量事实“，配合 pana 可以在本地复现评分。第二大脑可以这样搭：
• 底层（事实源）：dart analyze + DCM + pub deps + pana 的输出，全部从代码自动生成
• 中层（资产目录）：每个 Dart package 一份 catalog-info.yaml，接入 Backstage 统一管理，把 pana 评分、依赖图、lint 状态挂到对应服务的页面
• 顶层（架构陈述）：由于 Dart 项目多为 Flutter 应用 + 后端 service 混合，架构图建议用 Structurizr DSL 单独维护 C4 模型，与代码解耦
如果 Backstage 太重，轻量方案：用 dart doc 生成的 API 文档 + mdBook/Zola 静态站点，CI 自动把 pana 评分、依赖图、DCM 报表注入 Markdown。
三阶段落地路径
阶段一：体检（第 1 周）
一次跑齐：dart analyze、dart pub outdated、dart pub deps、pana、DCM 基础扫描。产出一份”系统健康画像“：lint 违规数、过时依赖、License 风险、复杂度热点。这份画像就是第二大脑的第一块砖。
阶段二：建边界（第 2–4 周）
• 用 Pub Workspaces 或 Melos 把项目拆成职责清晰的多个 package（core / domain / data / presentation 之类）
• 在 analysis_options.yaml 里启用 very_good_analysis 作为基线
• 用 custom_lint 写 2–3 条最关键的架构规则（跨层 import 禁令），先警告后阻断
• 接入 pana，所有内部包 pana 分数 ≥ 130 分
阶段三：守护 + 陈述（第 2 个月起）
• 所有工具进 CI，custom_lint 违规阻断合并
• Melos 统一版本发布流程，changelog 自动生成
• 第二大脑站点每周自动刷新，禁止手写架构图，全部从工具产出的事实自动渲染
与 Rust 生态的一个明显差异
Rust 的 lint/架构规则直接跑在编译器里（cargo clippy + Dylint），是”编译期强制“；Dart 的架构规则跑在 analyzer 上（custom_lint），是”IDE + CI 强制“。两者强度相当，但 Dart 的规则分发更方便——团队所有人 IDE 里实时看到违规并一键修复，开发体验通常比 Rust 顺滑。
一句话收束：Dart 治理的杠杆点是 analysis_options.yaml + custom_lint，把所有架构约定写进这一份 YAML 和少量 Dart 规则，用 Melos/Pub Workspaces 管住 monorepo，用 pana + DCM 把”健康“变成可对外陈述的数字——你的第二大脑同样会自己长出来。
