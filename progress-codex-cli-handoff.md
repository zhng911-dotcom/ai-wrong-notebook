# Codex CLI 交接文件 — Smart Wrong Notebook — 2026-05-18

## 项目定位

Smart Wrong Notebook 是 Flutter 移动应用，面向学生的 AI 错题本。一期重点是单机 Android。核心链路是：

> 拍照/框选图片 → AI 识别与解析 → 展示解析结果 → 生成举一反三 → 用户确认并保存到错题本

当前不要退回旧流程“拍照 → OCR 确认 → AI 分析”。

## 必守约束

- 不要 push 到 GitHub，除非用户明确要求。
- 不要 `git add .`。如需提交，只 stage 精确文件。
- 不要把 API Key 写入代码、日志、Markdown 或 commit。
- 不要硬编码某张 fixture 图的答案。
- 图形题读图不确定时，App 应显示“可能解法/需核对”，不要显示确定绿色答案。
- 不要修改 LaTeX 渲染引擎：
  - `lib/src/shared/widgets/math_content_view.dart`
  - `lib/src/shared/widgets/katex_math_view.dart`
  - `assets/katex/`

## 最新完成

- 真机反馈确认：错题本右上角 camera 入口已是“拍照 / 相册”。
- 继续加强 `generatedExercises` 质量门：
  - 拒绝“answer 指向的选项值”和 explanation 结论值明显冲突的练习题。
  - 典型覆盖：explanation 算出 `64-25π/2`，但 `answer=A` 指向 `64-25π`。
- 扩展本地图片 fixture 回归：
  - `test/tool/analyze_image_fixture_test.dart` 支持 `AI_FIXTURE_IMAGE`、`AI_FIXTURE_SET=local`、`AI_FIXTURE_CASES`。
  - qualityGate 区分 hard `issues` 和 soft `warnings`。
  - 文档：`docs/ai-fixture-regression.md`。
- framed semicircle fallback 从固定模板升级为轻量参数化：
  - 只覆盖“外框上/下边 + 右边高 + 左侧斜边为半圆直径 + 求半圆面积”这一窄 profile。
  - 解析源题上边、下边、高，生成同级题；简单/提高题用相邻变体。
  - 题目、选项、答案、解释、diagramData 使用同一组参数；图中目标标注为“求半圆面积”。
- 新 APK：
  - `build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260518-0150.apk`
  - SHA256：`8d439dc2878d9b9326689e99f5d064373d0e16872466eb195cc2b3d1a839f26b`

## 关键文件

- `lib/src/data/remote/ai/ai_analysis_service.dart`
  - AI prompt、JSON repair、generatedExercises 解析、质量门、fallback/profile。
  - 最新改动：answer/option/explanation 冲突门；framed semicircle 参数化 fallback。
- `test/data/remote/ai_analysis_service_test.dart`
  - generatedExercises 质量门、fallback、外角、framed semicircle 参数化回归。
- `test/tool/analyze_image_fixture_test.dart`
  - 本地图片 fixture 回归工具。
- `docs/ai-fixture-regression.md`
  - fixture 回归运行说明。
- `progress-current.md`
  - 最新 park 摘要。

## 验证结果

- `flutter test test/data/remote/ai_analysis_service_test.dart test/tool/analyze_image_fixture_test.dart test/features/analysis/exercise_practice_test.dart`
  - `EXIT_CODE=0`
  - `68 passed`, `1 skipped`
  - skipped 是真实图片 fixture 网络回归未设置环境变量。
- `flutter analyze --no-fatal-infos --no-fatal-warnings`
  - `EXIT_CODE=0`
  - 仍有 `100 issues found`，均 non-fatal info/warning。
  - 主要来自既有 LaTeX 渲染相关测试和未跟踪 geometry demo。
- `git diff --check`
  - `EXIT_CODE=0`
  - 通过。
- `flutter build apk --release`
  - `EXIT_CODE=0`
  - 构建成功，已复制 v62 时间戳 APK。

## 当前 Git 状态摘要

本 park 准备提交的精确文件清单：

- `lib/src/data/remote/ai/ai_analysis_service.dart`
- `test/data/remote/ai_analysis_service_test.dart`
- `test/tool/analyze_image_fixture_test.dart`
- `docs/ai-fixture-regression.md`
- `progress-current.md`
- `progress-codex-cli-handoff.md`

仍有大量既有未跟踪文件，不要 `git add .`。

## 风险点

- 真实模型批量 fixture 回归还没跑成：当前沙箱禁止通过 `/dev/tty` 隐藏读取 API key，不能安全注入 key。
- framed semicircle 参数化是窄范围 fallback，不是通用视觉纠错系统。
- 旧保存记录里的坏 diagramData 不会自动迁移。
- 本轮不 push。

## 下一步

1. 真机安装 `build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260518-0150.apk`。
2. 重点测：camera 入口、framed semicircle fallback 图文一致、AI 练习选项/解释冲突是否被替换。
3. 如需真实模型批量回归，在安全 shell 中设置环境变量后运行 `AI_FIXTURE_SET=local flutter test test/tool/analyze_image_fixture_test.dart`。
