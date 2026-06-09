# PersonSkills - 个人 AI Agent 技能库 🚀

本项目是为个人 AI 编码助手（如 Antigravity, Claude Code 等 Agent 框架）定制的技能（Skills）托管库。通过将特定的工作流规范、设计系统和测试框架沉淀为系统的 `SKILL.md`，能够使 Agent 在编写代码时严格遵循最佳实践，避免模板化的平庸产物。

---

## 📂 技能库目录架构

项目根目录下的 `skills/` 文件夹已被整理为扁平的双层结构，从而将自己创建的核心技能同他人仓库拉取的公开技能进行清晰分离：

```text
PersonSkills/
├── README.md               # 技能库总览与使用说明
└── skills/                 # 技能文件夹 (扁平化管理)
    ├── my-skills/          # 👤 个人自建核心技能 (Custom & Personal)
    │   ├── app-update-management/
    │   ├── legado-book-source-creation/
    │   └── novel-reader-integration/
    └── third-party/        # 📦 第三方及官方开源技能 (Third-party & Open source)
        ├── taste-skill/
        ├── react-patterns/
        ├── flutter-use-http-package/
        └── ... (33个公开的各类技术栈最佳实践)
```

---

## 👤 1. 个人自建核心技能 (`skills/my-skills/`)

这是您自己规划及从实际项目经验中提炼沉淀的核心业务场景技能：

*   **[novel-reader-integration](./skills/my-skills/novel-reader-integration/SKILL.md)**: 小说阅读核心技术整合。包含客户端自适应文本分页算法、文字转语音（TTS）播放进度同步、白噪音混音混合背景播放设计以及双层缓存（内存预取 + 离线本地 SQLite）策略。
*   **[legado-book-source-creation](./skills/my-skills/legado-book-source-creation/SKILL.md)**: Legado 书源规则设计与云端同步标准。包括 CSS 选择器调试避坑、MD5 散列主键生成公式、本地 SQLite 与云端 PostgreSQL (SSH + Docker 容器) 注入机制。
*   **[app-update-management](./skills/my-skills/app-update-management/SKILL.md)**: 移动应用发布与更新升级通知机制。规避客户端与服务端版本号冲突、包管理器防缓存配置及完整性校验。

---

## 📦 2. 第三方及官方开源技能 (`skills/third-party/`)

拉取自社区或官方团队的标准技术指南（由 UI/动效类、测试类、语言基础框架类等整合扁平化归入此目录）：

### 🎨 界面设计与动效 (UI & Motion)
*   **[taste-skill](./skills/third-party/taste-skill/SKILL.md)**: 拒绝平庸前端页面的视觉指南，包括留白比例、微排版与心理感知模型。
*   **[brutalist-skill](./skills/third-party/brutalist-skill/SKILL.md)**: 粗犷主义设计与强对比排版开发指南。
*   **[minimalist-skill](./skills/third-party/minimalist-skill/SKILL.md)**: 极简主义设计、克制的微交互及呼吸留白。
*   **[gpt-tasteskill](./skills/third-party/gpt-tasteskill/SKILL.md)**: AIDA 高效版式、无缝 Bento 网格与 GSAP 滚动动效。
*   **[soft-skill](./skills/third-party/soft-skill/SKILL.md)**: 柔和极光渐变与毛玻璃拟态（Glassmorphism）色盘规范。
*   **[redesign-skill](./skills/third-party/redesign-skill/SKILL.md)**: 既有项目重构视觉迭代开发标准。
*   **[stitch-skill](./skills/third-party/stitch-skill/SKILL.md)**: 通用流式多组件卡片协作架构。
*   **[brandkit](./skills/third-party/brandkit/SKILL.md)**: 核心品牌色盘设计令牌（Tokens）管理。

### 🧪 自动化测试与质量保障 (Testing & QA)
*   **[webapp-testing](./skills/third-party/webapp-testing/SKILL.md)**: 现代 Web E2E 高保真回归测试编写规范。
*   **[playwright-skill](./skills/third-party/playwright-skill/SKILL.md)**: Playwright 高级用例生成与多浏览器自动化执行。
*   **[android_ui_verification](./skills/third-party/android_ui_verification/SKILL.md)**: Android 模拟器 ADB 图形定位比对及完整性校验。

### 💻 核心语言与应用框架 (Development)
*   **[image-to-code-skill](./skills/third-party/image-to-code-skill/SKILL.md)**: 截图自动翻译高保真响应式页面的转化法则。
*   **[react-patterns](./skills/third-party/react-patterns/SKILL.md)**: React 解耦 Hooks 及复杂状态机编写规范。
*   **[nextjs-best-practices](./skills/third-party/nextjs-best-practices/SKILL.md)**: Next.js App Router 静态生成及服务器组件性能管理。
*   **Official Flutter 官方团队最佳实践包 (10 个技能)**: 包含 [flutter-apply-architecture-best-practices](./skills/third-party/flutter-apply-architecture-best-practices/SKILL.md) (分层架构设计)、[responsive-layout](./skills/third-party/flutter-build-responsive-layout/SKILL.md) (响应式适配)、集成与 Widget 单元测试、本地化 localization 等。
*   **Official Dart 官方团队最佳实践包 (9 个技能)**: 包含静态检查 [dart-run-static-analysis](./skills/third-party/dart-run-static-analysis/SKILL.md)、[pattern-matching](./skills/third-party/dart-use-pattern-matching/SKILL.md) (Dart 3 模式匹配解构) 等。

---

## 🚀 如何安装与使用

Agent 会在它的全局配置文件夹中扫描所有的 `SKILL.md` 文件。您可以直接将您需要的技能子文件夹复制到 Agent 的全局 `skills` 目录下：

*   **Antigravity / Gemini IDE 用户**:
    *   **Windows 路径**: `C:\Users\<你的用户名>\.gemini\config\skills\`
    *   **MacOS / Linux 路径**: `~/.gemini/config/skills/`

**Windows 示例命令**（复制 `novel-reader-integration` 自建技能到全局目录）：
```powershell
Copy-Item -Recurse -Force .\skills\my-skills\novel-reader-integration C:\Users\$env:USERNAME\.gemini\config\skills\
```
