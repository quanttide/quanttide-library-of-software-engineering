Go 治理三件套：golangci-lint（lint 总入口）+ go-arch-lint（架构边界守护）+ govulncheck（官方漏洞治理），再用 go mod graph + 可视化工具补齐依赖图，整个观测层就能完整外包。Go 生态的最大特点是官方工具链异常完整——go vet、go mod graph、govulncheck、go doc 都是官方出品，第三方工具只需要补”架构规则“和”度量“两块拼图，观测层的碎片化程度是四种语言里最低的。
Go 治理工具全景表
层级	解决什么问题	推荐工具	接入成本	开源状态	备注
官方静态检查	常见 bug、可疑代码	go vet	极低，官方内置	✅ 官方	go test 之前必跑的第一道闸门
深度静态分析	精准的 bug 检测、性能问题	staticcheck	低	✅ 开源	Go 社区最受尊敬的分析器，已集成到 gopls 和主流 IDE
Lint 总入口	统一调度 100+ linter	golangci-lint	低，YAML 一份	✅ 开源	事实标准，一个配置文件管所有 linter，支持插件机制
架构边界守护	分层规则、package 依赖禁令	go-arch-lint	中，YAML DSL	✅ 开源	Go 生态里最接近 ArchUnit 的工具，支持依赖层级、深度扫描、DI 检查
import 路径限制	禁止某些包被 import	depguard（golangci-lint 内置）	极低	✅ 开源	轻量方案，在 golangci-lint 里写几行 YAML 即可
自定义 Linter	团队特有规则	go/analysis 框架 + golangci-lint 插件	高，需写 Go 代码	✅ 官方框架	标准库级支持，比 Rust Dylint/TS custom_lint 更正统
模块依赖图	看清”谁依赖谁“	go mod graph + modgraphviz	极低	✅ 官方	go mod graph 官方输出，modgraphviz 转 Graphviz DOT
模块依赖可视化	Web UI 浏览依赖树	samber/go-mod-graph / modview	低	✅ 开源	Web 端可视化，比命令行直观得多
漏洞扫描	依赖中的已知 CVE	govulncheck	极低	✅ 官方	Go 官方安全团队出品，基于调用链分析（只在真正被调用时告警）
License 合规	依赖 License 白名单	go-licenses / go-license-detector	低	✅ 开源	配 GitHub Action 可自动化
包导入图（包级）	包间 import 关系	godepgraph	低	✅ 开源	输出 Graphviz DOT，粒度比 mod graph 更细（到包级）
全指标沉淀	统一报表、Quality Gate	SonarQube Community Build	中	✅ 社区版免费	官方支持 Go
完整工具清单可参考 analysis-tools.dev 收录的 118 个 Go 静态分析工具做二次筛选。
三个核心工具的深度说明
① golangci-lint —— Go 治理的统一容器
这是 Go 生态的”types cript-eslint + Knip + depguard“三合一。它把 staticcheck、govet、errcheck、gosec、revive、depguard 等上百个 linter 聚合到一份 .golangci.yml 里，一份配置文件 = 一套治理事实源，IDE 和 CI 完全一致。它还支持自定义 linter 插件机制——用官方 go/analysis 框架写好规则后，通过 golangci-lint custom 构建进二进制，团队约定和社区规则共用一个入口。这是 Go 治理里杠杆最高的一件。
② go-arch-lint —— Go 的”ArchUnit“
Go 官方没有内置架构边界机制（不像 TypeScript 的 Project References），但 go-arch-lint 把这块补齐了。它用一个 YAML DSL 声明”哪个 package 可以依赖哪个 package“，支持分层、依赖层级、深度方法调用扫描甚至 DI 关系检查，CI 一条命令就能阻断违规合并。相比手工维护架构图，这份 YAML 本身就是可执行的架构陈述——它就是你的架构图，也是你的架构测试。
③ govulncheck —— 官方供应链治理
Go 官方安全团队出品的漏洞扫描器，独特之处在于基于调用链分析：只在漏洞真正被你的代码调用到时才告警，比 npm audit/cargo audit 的”依赖命中就报“精准得多，告警噪音小到可以真的每个都处理。它和 go mod graph、go-licenses 一起构成了 Go 供应链治理的官方三件套。
第二大脑载体
• 底层（事实源）：go vet + staticcheck + golangci-lint 报表 + go mod graph 输出 + govulncheck 报告 + go-arch-lint 验证结果，全部从代码自动生成
• 中层（资产目录）：每个 Go module 一份 catalog-info.yaml 接入 Backstage，把 SonarQube 报表、go-mod-graph 可视化、go-arch-lint 规则集挂到对应服务页面
• 顶层（架构陈述）：go-arch-lint 的 YAML 本身就可以渲染成架构图（它支持可视化输出），配合 Structurizr DSL 维护更宏观的 C4 模型
如果 Backstage 太重，最低成本方案：pkgsite（Go 官方文档站，可私有部署）+ mdBook/Zola 静态站点，CI 把 golangci-lint 报表、go-mod-graph SVG、go-arch-lint 架构图注入 Markdown。
三阶段落地路径
阶段一：体检（第 1 周）
一次跑齐：go vet、staticcheck、golangci-lint run、go mod graph、govulncheck ./...、go-licenses check ./...。产出一份”系统健康画像“：lint 违规数、可疑代码热点、模块依赖图、已知漏洞清单、License 风险。这份画像是第二大脑的第一块砖。
阶段二：建边界（第 2–4 周）
• 用 Go Workspaces（go.work）或多 module 结构把项目拆成职责清晰的模块
• 用 go-arch-lint 写 3–5 条核心架构规则（分层禁令、领域层不得依赖基础设施层），先报告模式后阻断模式
• 在 .golangci.yml 里启用 depguard 做轻量级 import 限制，与 go-arch-lint 形成软硬互补
• 全部规则进 CI，违规阻断合并
阶段三：守护 + 陈述（第 2 个月起）
• golangci-lint + go-arch-lint + govulncheck 全部强制阻断
• SonarQube Community Build 作为统一 Quality Gate
• 第二大脑站点每周自动刷新，架构图一律从 go-arch-lint/go-mod-graph 产出，禁止手写
与其他三种语言的对比一句话
Rust 靠编译器 + cargo 原生守护、Dart 靠一份 analysis_options.yaml、TypeScript 靠 ESLint 当治理容器，而 Go 的独特优势是”官方工具链已经覆盖了 70% 的治理面“——go vet/go mod graph/govulncheck/go doc 全是官方出品，你只需要拼装 golangci-lint（lint 聚合）和 go-arch-lint（架构规则）两件第三方工具就能把治理闭环跑通。代价是 Go 缺少类似 ArchUnit/dependency-cruiser 那样成熟度极高的第三方架构治理生态，go-arch-lint 相对年轻，复杂规则可能需要自己写 go/analysis 插件补位。
一句话收束：Go 治理的杠杆点是 golangci-lint 的一份配置 + go-arch-lint 的一份 YAML，前者把所有 lint/安全/风格规则统一进 CI，后者把架构边界写成可执行的 DSL——官方工具链已经替你做完观测，你的第二大脑只需要一个静态站点把它呈现出来。
