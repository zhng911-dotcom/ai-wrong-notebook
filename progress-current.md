# 项目进展 / Park — 2026-05-18

## Done

- 真机反馈确认：错题本右上角 camera 入口已是“拍照 / 相册”，不再直接复用旧图。
- 继续加强 AI 举一反三质量门：
  - `generatedExercises` 若 `answer` 指向的选项值与 `explanation` 算出的结论明显不同，且 explanation 结论值出现在其他选项中，会被拒绝并用 fallback 补齐。
  - 覆盖典型坏题：解释算出 `64-25π/2`，但 `answer=A` 指向 `64-25π`。
- 扩展图片 fixture 回归工具：
  - `test/tool/analyze_image_fixture_test.dart` 支持单张 `AI_FIXTURE_IMAGE`、默认批量 `AI_FIXTURE_SET=local`、显式 JSON 批量 `AI_FIXTURE_CASES`。
  - qualityGate 区分 hard `issues` 与 soft `warnings`；图形读图 `needsReview` 但内部一致时仅 warning，不 fail。
  - 新增文档：`docs/ai-fixture-regression.md`。
- framed semicircle fallback 从固定模板升级为轻量参数化：
  - 只覆盖“外框上/下边 + 右边高 + 左侧斜边为半圆直径 + 求半圆面积”这一窄 profile。
  - 解析源题上边、下边、高，生成同级题；简单/提高题用相邻变体。
  - 题目、选项、答案、解释、diagramData 使用同一组参数；图中目标标注为“求半圆面积”，不再误写“求此区域”。
- 构建新 APK：
  - `build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260518-0150.apk`
  - 大小：`68M`
  - SHA256：`8d439dc2878d9b9326689e99f5d064373d0e16872466eb195cc2b3d1a839f26b`

## Verification

- `flutter test test/data/remote/ai_analysis_service_test.dart test/tool/analyze_image_fixture_test.dart test/features/analysis/exercise_practice_test.dart`
  - `EXIT_CODE=0`
  - `68 passed`, `1 skipped`
  - skipped 是真实图片 fixture 网络回归未设置 `AI_FIXTURE_IMAGE` / `AI_FIXTURE_CASES` / `AI_FIXTURE_SET=local`，符合预期。
- `flutter analyze --no-fatal-infos --no-fatal-warnings`
  - `EXIT_CODE=0`
  - 仍有 `100 issues found`，均为 non-fatal info/warning。
  - 主要来自既有 `lib/src/shared/widgets/math_content_view.dart`、`test/shared/widgets/math_content_view_test.dart`、未跟踪 `test/tool/geometry_canvas_demo.dart`。
  - 未处理，继续遵守“不碰 LaTeX 渲染引擎”的约束。
- `git diff --check`
  - `EXIT_CODE=0`
  - 通过。
- `flutter build apk --release`
  - `EXIT_CODE=0`
  - 构建成功：`build/app/outputs/flutter-apk/app-release.apk (71.6MB)`。

## Changed Files For WIP Commit

- `lib/src/data/remote/ai/ai_analysis_service.dart`
- `test/data/remote/ai_analysis_service_test.dart`
- `test/tool/analyze_image_fixture_test.dart`
- `docs/ai-fixture-regression.md`
- `progress-current.md`
- `progress-codex-cli-handoff.md`

## Blockers / Risks

- 真实模型批量 fixture 回归还没跑成：当前沙箱禁止通过 `/dev/tty` 隐藏读取 API key，不能安全注入 key。不要把 API key 写入文件、日志、Markdown 或 commit。
- 本轮没有 push。
- 工作区仍有大量既有未跟踪文件；不要 `git add .`。
- framed semicircle 参数化是窄范围 fallback，不是通用视觉纠错系统。
- 旧保存记录里的坏 diagramData 不会自动迁移。

## Next First Step

1. 安装并真机测试：`build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260518-0150.apk`。
2. 重点看：
   - camera 入口仍为“拍照 / 相册”。
   - framed semicircle fallback 题目和图中目标一致，图里标“求半圆面积”。
   - AI 生成练习题若选项/解释冲突，应被质量门替换。
3. 如需真实模型批量回归，在安全 shell 中设置环境变量后运行 `AI_FIXTURE_SET=local flutter test test/tool/analyze_image_fixture_test.dart`。

## Tomorrow First Action

- 从真机测试 `ai-wrong-notebook-v62-20260518-0150.apk` 开始；如果通过，再考虑是否 push 或继续做 PDF/练习质量扩展。

Good night.
