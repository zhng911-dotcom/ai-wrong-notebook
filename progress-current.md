# 项目进展 / Park — 2026-05-13

## 当前版本状态

- 当前 APK：`build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260513-0325.apk`
- 当前版本号仍为 v62：本次属于 bug 修复与验证包，小改动只更新时间戳，不递增版本号。
- APK 已按项目规则留在 `build/app/outputs/flutter-apk/`，未复制到 Desktop。
- 本地仍有多组未提交改动；提交前需要显式确认文件清单，不能 `git add .`。

## 今日完成

### 1. 继续排查几何练习题 `diagramData` 首次不显示问题

用户真机验证确认：

- 拍照 → AI 解析 → 解析结果页 → 立即点击「开始练习」：举一反三几何题无配图。
- 首次保存到错题本 → 直接进入错题详情 → 立即练习：举一反三几何题无配图。
- 保存后重新从「错题本 → 错题详情 → 开始练习」进入：同一题有配图。

关键结论：持久化 JSON 中 `savedExercises[].diagramData` 是存在的，问题不在生成或 SharedPreferences 序列化丢字段；差异在“即时内存对象”和“重新从 JSON 反序列化对象”的结构/解析路径。

### 2. 第一层修复：练习页缓存刷新

文件：`lib/src/features/analysis/presentation/exercise_practice_screen.dart`

- 原逻辑只按 `questionId` 判断是否复用 `_exercises`。
- 已改为同时比较：
  - `questionId`
  - `practiceContext.candidateId`
  - 练习数据源版本（包含 `diagramData`）
- 避免同一题数据补全后仍复用旧 `_exercises`。
- 完成练习/继续练习时同步内部 source version，避免完成页被误重置。

### 3. 第二层修复：几何图组件兼容即时内存 Map

文件：`lib/src/features/analysis/presentation/widgets/geometry_diagram_widget.dart`

- `GeometryDiagramWidget` 原解析器对 `elements` / `auxiliaryLines` 子元素使用严格 `Map<String, dynamic>` cast。
- 即时 AI 内存对象里的嵌套 map 可能是 `Map<dynamic, dynamic>`，而重新进入错题本后 JSON 反序列化会变成更稳定的字符串 key map。
- 已新增 `_asStringMap`，解析 `elements` / `auxiliaryLines` 时兼容 `Map` 并转换为 `Map<String, dynamic>`。
- 这次没有改 `math_content_view.dart`、`katex_math_view.dart`、`assets/katex/`，也没有动 LaTeX 正则或 LaTeX 渲染引擎。

### 4. 测试与验证

- `flutter test test/features/analysis/exercise_practice_test.dart` → `EXIT_CODE:0`
  - 覆盖同一题 `diagramData` 从无到有时练习页刷新。
  - 覆盖 `GeometryDiagramWidget` 解析即时内存 `Map<dynamic, dynamic>` 子元素。
- `flutter analyze --no-fatal-infos lib/src/features/analysis/presentation/widgets/geometry_diagram_widget.dart test/features/analysis/exercise_practice_test.dart lib/src/features/analysis/presentation/exercise_practice_screen.dart` → `EXIT_CODE:0`
- LaTeX 渲染相关文件 diff 检查 → `EXIT_CODE:0`，无输出。
- `flutter build apk --release` → `EXIT_CODE:0`
- 最新 APK：`build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260513-0325.apk`

## Blockers / 风险点

1. 本地没有 Android 真机或模拟器，无法由我直接跑完整 UI 手动流程；需要用户用 `0325` APK 真机复测两个即时入口。
2. 真机旧包 `0242` 已证明第一层缓存修复不够；`0325` 包加入了几何图内存 Map 兼容修复，仍待真机确认。
3. `ai_analysis_service.dart` 里 AI prompt 文本包含 LaTeX 反斜杠，完整 analyzer 会报 `unnecessary_string_escapes` info；本次按用户要求没有清理这些 prompt 文本，更没有触碰 LaTeX 渲染引擎。
4. 当前 git working tree 有较多历史/并行改动与未跟踪文件，提交必须显式确认文件清单，避免误提交无关草稿。

## Next First Step

1. 用户安装并测试 `ai-wrong-notebook-v62-20260513-0325.apk`：
   - AI 解析后立即「开始练习」是否有图。
   - 首次保存到错题详情后立即练习是否有图。
   - 重新从错题本进入是否仍正常有图。
2. 如果 `0325` 仍没图，下一步加最小 debug：在练习页输出当前 exercise 的 `diagramData.runtimeType`、`elements.runtimeType`、首个 element runtimeType，以及 `GeometryDiagramWidget` parse result；用真机 logcat / flutter logs 看即时入口和重进入口差异。
3. 完成 park 本地 WIP 提交：先确认精确文件清单，再 `git add <explicit files>`，commit message 固定 `wip: end of day state`。

## Tomorrow first action

- 先看用户真机复测 `0325` APK 结果；如果通过，再收尾提交；如果失败，按上述 debug 点继续定位即时内存对象和图形 parser 的差异。
