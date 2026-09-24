TypeScript 治理三件套：typescript-eslint（强制 lint）+ dependency-cruiser（架构边界守护）+ Knip（依赖与死代码清理），再用 Madge 做循环依赖可视化、Sherif/Turborepo 管 monorepo，整个观测层就能完整外包出去。TypeScript 生态的独特优势是——ESLint 是架构治理的万能容器，从代码风格到跨层依赖规则到 dead code 检测，都能用一份 eslint.config.js 统一管住，这比 Java/Rust 需要拼装多个工具要清爽。
TypeScript 治理工具全景表
层级	解决什么问题	推荐工具	接入成本	开源状态	备注
官方编译器	类型错误、严格模式	tsc —strict + tsc —noEmit	极低，SDK 内置	✅ 官方	把 ”strict“: true 打开就是第一道闸门
强制 Lint	代码风格、类型感知规则	typescript-eslint	低，扁平配置一份	✅ 开源	事实标准，配合 eslint-config-next/standard-with-typescript 起步
架构边界守护	跨层/跨模块 import 规则	dependency-cruiser 或 eslint-plugin-boundaries	中	✅ 开源	dependency-cruiser 独立工具更强大灵活，boundaries 嵌入 ESLint 更顺手
简单 import 限制	禁止某些路径被 import	no-restricted-imports / import/no-restricted-paths	极低	✅ 官方规则	不需要完整依赖图时最轻量
循环依赖检测	破坏 import 环	Madge / dpdm	低	✅ 开源	Madge 可生成 SVG 可视化依赖图
死代码/未用导出	未使用文件、export、依赖	Knip	低	✅ 开源	已取代 ts-prune，150+ 框架插件，monorepo 感知
依赖清理	未使用的 npm 依赖	Knip + depcheck	低	✅ 开源	Knip 已覆盖大部分场景
依赖图可视化	看清”谁依赖谁“	dependency-cruiser / Madge	低	✅ 开源	都能输出 Mermaid/DOT/SVG，可直接嵌文档站
AST 深度分析	自定义规则、大规模重构	ts-morph	高，需写代码	✅ 开源	TypeScript Compiler API 的友好封装，可编程分析/重构
License/供应链合规	License 白名单、漏洞	license-checker + npm audit	低	✅ 开源	企业级可选 FOSSA
Monorepo 编排	多 package 构建/缓存/任务	Turborepo / Nx	中	✅ 开源	Turborepo 轻量、Nx 功能全（含依赖图 UI）
Monorepo 一致性	依赖版本、package.json 规范	Sherif	低，零配置	✅ 开源	意见化 monorepo linter，自动发现版本不一致
包管理器工作区	多 package 依赖提升	pnpm workspaces	低	✅ 开源	目前 TS 社区事实标准
官方边界机制	编译器级项目隔离	TypeScript Project References	中	✅ 官方	只有被显式 reference 的项目才能 import，是编译器级的架构守护
全指标沉淀	统一报表、Quality Gate	SonarQube Community Build	中	✅ 社区版免费	官方支持 TS，数百条规则
三个核心工具的深度说明
① dependency-cruiser —— TypeScript 的”ArchUnit“
这是 TS 治理里最值得投入的一件。它不只是画依赖图，核心能力是把架构规则写成可执行的验证器：分层规则（presentation → application → domain → infrastructure）、禁止跨 feature import、禁止绕过 index.ts 的公共 API 直接 import 内部文件、禁止从 apps/ 反向 import packages/ 等等。规则写在 .dependency-cruiser.cjs 里，CI 一条 depcruise —validate 就能阻断违规合并。相比 eslint-plugin-boundaries（同能力但嵌在 ESLint 里），dependency-cruiser 规则表达更丰富，还能做”validate 只查不阻断“的报告模式。
② Knip —— 死代码与依赖治理的瑞士军刀
 Knip 的价值远超”找 dead code“——它会基于真实入口点（如 Next.js 路由、Vite 入口、Jest 配置）反向分析，精准识别：未使用的文件、未使用的 export、未使用的 dependencies、未使用的 devDependencies、重复的 workspace 依赖。配合 150+ 框架插件，它对 monorepo 场景的开箱即用性远超同类工具，是第二大脑里”代码健康“维度的主力。
③ TypeScript Project References —— 编译器级的边界守护
这是 TS 官方给出的架构隔离机制：只有被 references 显式声明的项目才能被 import，其他一律编译失败。它和 dependency-cruiser 是互补关系——Project References 是编译器硬边界，dependency-cruiser 是规则层软约束（能表达更细的语义如”只允许 import index.ts“）。大型项目建议两层都上，monorepo.tools 上有详细实践。
第二大脑载体
• 底层（事实源）：tsc 严格模式报告 + typescript-eslint 报表 + dependency-cruiser 依赖图与违规清单 + Knip 死代码清单 + Madge SVG 架构图，全部从代码自动生成
• 中层（资产目录）：每个 package 一份 catalog-info.yaml 接入 Backstage，把 SonarQube 报表、dependency-cruiser 图、Knip 报告挂到对应服务页面
• 顶层（架构陈述）：Structurizr DSL 维护一份 C4 模型，对外架构图全部从它渲染
如果 Backstage 太重，最低成本方案：Turborepo 自带的依赖图 UI + mdBook/Zola 静态站点，CI 把 dependency-cruiser 的 Mermaid 输出和 Knip 报告注入 Markdown，零运维。
三阶段落地路径
阶段一：体检（第 1 周）
一次跑齐：tsc —noEmit —strict、eslint + typescript-eslint、knip、madge —circular、license-checker。产出一份”系统健康画像“：类型错误数、lint 违规、循环依赖清单、死代码占比、License 风险。这份画像是第二大脑的第一块砖。
阶段二：建边界（第 2–4 周）
• 用 pnpm workspaces + Turborepo（或 Nx）拆成职责清晰的多个 package
• 启用 TypeScript Project References 建立编译器级硬边界
• 用 dependency-cruiser 写 3–5 条核心架构规则（分层禁令、公共 API 强制），先报告模式后阻断模式
• 接入 Sherif 保证 monorepo 依赖版本一致
• Knip 在 CI 定期跑，死代码/未用依赖逐次清理
阶段三：守护 + 陈述（第 2 个月起）
• 所有工具进 CI，dependency-cruiser 违规阻断合并
• SonarQube Community Build 作为统一 Quality Gate
• 第二大脑站点每周自动刷新，架构图一律从 dependency-cruiser/Structurizr 产出，禁止手写
与 Rust/Dart 生态的对比一句话
Rust 靠编译器和 cargo 生态原生守护，Dart 靠 analysis_options.yaml 一份文件管所有，TypeScript 则是把 ESLint 当作治理容器——所有规则都能写进同一份 eslint.config.js，但代价是需要拼装多个插件。TypeScript 生态工具密度最高、迭代最快，但也最碎片化，选型时优先用”事实标准“（typescript-eslint、dependency-cruiser、Knip、Turborepo）就能覆盖 90% 场景。
一句话收束：TypeScript 治理的杠杆点是 dependency-cruiser 的规则文件 + Project References 的硬边界，把所有架构约定写成可执行验证，用 Knip 持续瘦身，用 Turborepo/Nx 管住 monorepo——你的第二大脑同样从 CI 自动长出来。
