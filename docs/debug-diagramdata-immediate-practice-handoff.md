# diagramData 即时练习无图交接 — 2026-05-13

## 背景

目标问题：几何题的举一反三练习题里，`diagramData` 已生成并保存，但部分入口首次进入练习页不显示配图。

用户真机反馈（APK `ai-wrong-notebook-v62-20260513-0242.apk`）：

| 入口 | 结果 |
|---|---|
| 拍照 → AI 解析 → 解析结果页 → 立即「开始练习」 | 无图 |
| 首次保存到错题本 → 直接进入错题详情 → 立即练习 | 无图 |
| 保存后重新从「错题本 → 错题详情 → 开始练习」进入 | 有图 |

用户提供的 SharedPreferences JSON 里，`savedExercises[].diagramData` 明确存在，第二题也有完整 `elements` 和 `auxiliaryLines`。

## 已排除

1. **持久化字段缺失**：排除。JSON 中有 `savedExercises[].diagramData`。
2. **模型 `GeneratedExercise.copyWith()` 丢字段**：当前是 sentinel 模式，不传 `diagramData` 时保留原值。
3. **练习页条件完全错误**：练习页确实有 `if (exercise.diagramData != null) GeometryDiagramWidget(...)`。
4. **LaTeX 渲染引擎相关**：本轮没有修改 `math_content_view.dart`、`katex_math_view.dart`、`assets/katex/` 或 LaTeX 正则。

## 当前判断

关键差异是：

- 即时入口使用内存里的 `QuestionRecord` / `GeneratedExercise`。
- 重新从错题本进入使用 SharedPreferences JSON 反序列化后的对象。

因此问题不是“有没有 diagramData”，而更可能是即时内存对象中的 `diagramData` 嵌套结构类型与 JSON 反序列化后的类型不完全一致，导致 `GeometryDiagramWidget` parser 在即时入口失败。

## 今日代码改动

### 1. 练习页缓存刷新

文件：`lib/src/features/analysis/presentation/exercise_practice_screen.dart`

新增/调整：

- `_practiceCandidateId`
- `_exerciseSourceVersion`
- `_exerciseSourceVersionOf(...)`
- `_exerciseVersion(...)`

目的：不再只按 `questionId` 复用 `_exercises`，同题但练习数据源变化（包括 `diagramData` 变化）时重新构建练习列表。

同时在 `_finishRound` / `_continuePractice` 写回 provider 前同步 `_exerciseSourceVersion`，避免内部写回导致完成页被重置。

### 2. 几何图 parser 兼容内存动态 Map

文件：`lib/src/features/analysis/presentation/widgets/geometry_diagram_widget.dart`

修复点：

- 原代码：`_parseEl(item as Map<String, dynamic>)`
- 新代码：通过 `_asStringMap(item)` 兼容 `Map<String, dynamic>` 和普通 `Map`，再 `Map<String, dynamic>.from(value)`。
- `elements` 和 `auxiliaryLines` 都已处理。

动机：即时 AI 内存对象中的嵌套 map 可能不是严格 `Map<String, dynamic>`；重新 JSON 加载后能正常，所以需要 parser 接受更宽的 Map 类型。

### 3. 测试

文件：`test/features/analysis/exercise_practice_test.dart`

新增覆盖：

1. `refreshes practice exercises when diagramData appears for same question`
   - 模拟同一题先无图、后补上 `diagramData`。
   - 断言练习页出现 `GeometryDiagramWidget`。

2. `renders diagramData with dynamic map elements from memory`
   - 构造 `diagramData`，其中 `elements` 子项使用 `Map<Object?, Object?>`。
   - 断言 `GeometryDiagramWidget` 能渲染出 `CustomPaint`。

## 已运行验证

```bash
flutter test test/features/analysis/exercise_practice_test.dart
# EXIT_CODE:0
```

```bash
flutter analyze --no-fatal-infos lib/src/features/analysis/presentation/widgets/geometry_diagram_widget.dart test/features/analysis/exercise_practice_test.dart lib/src/features/analysis/presentation/exercise_practice_screen.dart
# EXIT_CODE:0
```

```bash
git diff --name-only -- lib/src/shared/widgets/math_content_view.dart lib/src/shared/widgets/katex_math_view.dart assets/katex
# EXIT_CODE:0，无输出
```

```bash
flutter build apk --release
# EXIT_CODE:0
```

最新 APK：

```text
build/app/outputs/flutter-apk/ai-wrong-notebook-v62-20260513-0325.apk
```

## 待用户真机复测

使用 `0325` APK 测：

1. 拍照 → AI 解析 → 解析结果页 → 立即点击「开始练习」：几何图是否显示。
2. 首次保存到错题本 → 直接进入错题详情 → 立即练习：几何图是否显示。
3. 保存后重新从错题本进入：是否仍显示。

## 如果仍失败，下一步

加最小 debug，不改 LaTeX：

- 在练习页 `exercise.diagramData != null` 前打印：
  - `exercise.diagramData.runtimeType`
  - `exercise.diagramData?['elements'].runtimeType`
  - `first element runtimeType`
  - `exercise.id` / `roundIndex` / `sourceExerciseId`
- 在 `GeometryDiagramWidget.tryFromJson` 打印 parser 是否失败，以及失败原因。
- 用真机 `flutter logs` 或 logcat 对比：
  - 即时解析入口
  - 首次保存后详情入口
  - 重新从错题本入口

## 注意事项

- 用户多次强调：不要碰 LaTeX 渲染引擎。
- 不要修改 `lib/src/shared/widgets/math_content_view.dart`、`lib/src/shared/widgets/katex_math_view.dart`、`assets/katex/` 或 LaTeX 正则，除非用户明确要求。
- APK 命名规则：`ai-wrong-notebook-v62-YYYYMMDD-HHmm.apk`，输出目录固定 `build/app/outputs/flutter-apk/`，不复制到 Desktop。
- 本地提交前必须显式确认文件清单，不能 `git add .`。
