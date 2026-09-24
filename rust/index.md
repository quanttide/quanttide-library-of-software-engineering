直接给你一套 Rust 治理的”入门三件套“：Clippy（强制 lint）+ cargo-deny（依赖治理）+ SonarQube Community Build（指标沉淀），先用这三样把观测外包出去，你的精力再放在消费它们的事实上。Rust 生态的工具密度其实比 Java/JS 高得多——因为 cargo 体系本身就内置了很多治理能力，关键是把它们串成流水线，而不是一个个孤立地跑。
Rust 开源治理工具全景表
按你要治理的对象从细到粗分四层，全部为开源或社区版可用：
层级	解决什么问题	推荐工具	GitHub 地址	接入成本	备注
语言级强制 Lint	代码风格、惯用法、潜在 bug	Clippy + rustfmt	rust-lang/rust-clippy	极低，cargo clippy 一行	所有 Rust 项目的第一道闸门，把 -D warnings 打进 CI
自定义 Lint	团队特有的架构/编码约定	Dylint	trailofbits/dylint	中，需写 Rust lint	把”我们公司不许这么写“变成可执行 lint，比 code review 稳定
深度静态分析	数据流、安全漏洞	Semgrep	semgrep/semgrep	低，YAML 规则	Rust 已 GA，支持跨函数数据流分析和 40+ Pro 规则
复杂度/度量	圈复杂度、嵌套深度、函数规模	rust-code-analysis（Mozilla）	mozilla/rust-code-analysis	中，作为库/CLI 使用	支持多语言，可编程提取度量指标
热点/腐化分析	”哪里最该治理“	CodeScene（社区版免费）	codescene/codescene	低	官方专门扩展了 Rust 的 Code Health 配置
依赖清理	未使用的依赖	cargo-machete + cargo-udeps + cargo-neat	bnjbvr/cargo-machete	低	machete 用文本匹配快、udeps 用编译器精准、neat 补足 workspace 场景
依赖安全/合规	RustSec 漏洞、License、来源	cargo-deny + cargo-audit	EmbarkStudios/cargo-deny	低	cargo-deny 覆盖了 audit 的全部能力，还加了 License/来源/重复版本管控，是被低估的工具
Unsafe 代码管控	unsafe 块审计	cargo-geiger	rust-secure-code/cargo-geiger	低	输出 unsafe 密度报告，安全敏感场景必备
Workspace 依赖治理	多 crate 依赖去重、特性统一	guppy + cargo hakari	guppy-rs/guppy	中	Meta/Graphite 出品，大规模 workspace 必备；hakari 统一第三方依赖 feature 集
API 兼容性	SemVer 违规检测	cargo-semver-checks	obi1kenobi/cargo-semver-checks	低	用编译器级精度检测 API 破坏性变更
供应链信任	依赖审计与评审记录	cargo-vet（Mozilla）	mozilla/cargo-vet	中	建立”每个依赖都被审计过“的可追溯记录
模块结构可视化	crate 内部模块图	cargo-modules	regexident/cargo-modules	低	把 crate 的模块树/依赖图渲染成 Graphviz/Mermaid，立刻看清”谁依赖谁“
Workspace 依赖图	crate 间依赖可视化	cargo-tree + cargo-depgraph	内置于 cargo	极低	cargo tree 是官方工具，cargo tree -d 直接看重复依赖
全指标沉淀	统一报表、Quality Gate	SonarQube Community Build	SonarSource/sonarqube	中	官方已支持 Rust；Community Build 是免费开源版本
代码知识图谱	大规模跨仓库分析	Joern	joernio/joern	高	Code Property Graph 理念，但 Rust 前端支持相对有限，小公司初期不必上
完整工具清单可以参考 analysis-tools.dev 收录的 75 个 Rust 静态分析工具做二次筛选。
三个核心工具的深度说明
① Clippy + Dylint —— 把架构约定变成可执行规则
Rust 的独特优势是 lint 系统极其强大。Clippy 自带 700+ 规则，而 Dylint 允许你写团队自定义 lint（比如”禁止在 domain crate 里引入 sqlx“）。这两样组合起来，相当于 Java 世界的 ArchUnit，但更轻量——它们直接在编译期介入，绕不过去。建议在 CI 里以 cargo clippy — -D warnings 强制阻断，自定义 lint 走 Dylint 单独发布。
② cargo-deny —— 一个工具管住整个依赖面
这是 Rust 治理里性价比最高的一件。它同时管四件事：RustSec 安全公告、License 白名单、依赖来源（比如禁用 git 依赖）、重复版本去重。用一份 deny.toml 描述规则，CI 一条命令出报告，这份报告本身就是你对外陈述”我们的依赖是干净的“的可审计事实。配合 cargo-vet，还能补上”每个依赖都经过人审“的追溯链。
③ cargo-modules + cargo-tree —— Rust 治理的”架构可视层“
这是你”第二大脑“的视觉底座。cargo-modules 把单个 crate 内部的模块结构渲染成图，cargo-tree 把 workspace 里多个 crate 的依赖关系渲染成图。两者加起来就能产出”系统架构图“和”模块依赖图“，而且是从代码直接生成、永远不腐烂的——这正是”准确对外陈述业务信息“的根基。建议每 CI 运行一次，输出到 Mermaid/Graphviz，再嵌入到文档站。
第二大脑载体：Backstage 还是 rustdoc + Structurizr
Rust 生态有一个天然优势——rustdoc 本身就是从代码生成的文档系统，比 Java 的 Javadoc 质量高一个量级。所以你的第二大脑可以分层搭建：
• 底层（事实源）：rustdoc + cargo-modules + cargo-tree 的输出，全部从代码自动生成
• 中层（资产目录）：Backstage 用 catalog-info.yaml 描述每个 crate 的 Owner、用途、依赖关系，把 SonarQube 报表、cargo-deny 报表、cargo-modules 架构图都挂到对应服务的页面上，形成”一个 crate 一页纸“
• 顶层（架构陈述）：Structurizr DSL 用一份 DSL 文件描述整个系统的 C4 模型，所有对外架构图都从它渲染，不会和代码脱节
如果 Backstage 对小公司太重，最低成本方案是：用 mdBook 或 Zola 搭静态站点，CI 自动把 cargo-modules 图、cargo-deny 报表、SonarQube 指标注入 Markdown，零运维成本就能跑起来。
三阶段落地路径（针对 Rust 项目）
阶段一：体检（第 1–2 周）
一次性接入 cargo clippy、cargo machete、cargo deny、cargo geiger，全仓库跑一遍。产出物是一份”系统健康画像“：lint 违规清单、未使用依赖、License 风险、unsafe 密度、依赖重复版本。这份画像就是你第二大脑的第一块砖。
阶段二：建边界（第 3–6 周）
• 用 cargo workspace 把单仓库拆成职责清晰的多个 crate（Rust 社区在这个话题上最权威的参考是 matklad 的《Large Rust Workspaces》，他是 rust-analyzer 作者，讲的是 rust-analyzer 这种百万行级 workspace 的真实经验）
• 用 Dylint 把 crate 边界规则写成自定义 lint（比如”web 层不得直接依赖 repository crate 的私有模块“）
• 用 cargo hakari 统一第三方依赖的 feature 集，避免编译矩阵爆炸
• 用 cargo-semver-checks 守住对外 API 的 SemVer 承诺
阶段三：守护 + 陈述（第 2 个月起）
• 所有工具接入 CI，违规阻断合并
• SonarQube Community Build 作为统一指标面板，设定 Quality Gate
• 第二大脑站点（Backstage 或静态站）每周自动刷新，对外陈述信息永远从工具产出的事实自动生成，禁止手写任何会腐烂的架构图
一句话收束
Rust 治理的独特优势是：语言和 cargo 生态已经把”观测“做成了基础设施（lint、依赖审计、模块图、SemVer 检查全是官方或半官方工具），你唯一要做的就是用 CI 把它们串成一条流水线，把产出的报表喂进一个静态站点或 Backstage——你的”第二大脑“会自己长出来，不需要你造。