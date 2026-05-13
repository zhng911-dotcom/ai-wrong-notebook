# diagramData 不显示 Bug 调查交接

## Bug 现象

几何题的 `diagramData`（结构化几何图形 JSON）在练习页面（举一反三）不显示：

| 场景 | 结果 |
|------|------|
| AI 解析后 → 直接开始练习 | 无图 |
| 先保存到错题本 → 从详情页练习 | 无图 |
| 之后重新进入错题本/复习 → 练习 | 有图 |

用户已确认：SharedPreferences 中的 JSON 数据里 `savedExercises[].diagramData` 字段是存在的。

## 已排除的原因

1. **`GeneratedExercise.copyWith()` 丢失 diagramData** — 已确认使用 sentinel 模式，不传 diagramData 时保留原值（`generated_exercise.dart:125,147-150`）

2. **`_nextPracticeRound()` 丢失 diagramData** — 该方法只 copyWith id/questionId/order/isCorrect/userAnswer/roundIndex 等，不传 diagramData，所以保留（`exercise_practice_screen.dart:576-589`）

3. **渲染逻辑问题** — 渲染代码正确检查 `exercise.diagramData != null` 并显示 `GeometryDiagramWidget`（`exercise_practice_screen.dart:185-192`）

4. **序列化/反序列化丢失** — `toJson()`/`fromJson()` 链路完整：
   - `GeneratedExercise.toJson()` 包含 diagramData（line 91）
   - `GeneratedExercise.fromJson()` 通过 `_parseDiagramDataField` 解析（line 55-56）
   - `QuestionRecord.toJson()/fromJson()` 正确序列化 savedExercises
   - `CandidateAnalysisSnapshot.toJson()/fromJson()` 也正确处理

5. **数据库层丢失** — 实际使用的是 `SharedPrefsQuestionRepository`（不是 drift），直接 JSON 序列化整个 QuestionRecord

## 关键代码路径

### 练习页加载逻辑 (`exercise_practice_screen.dart`)

```dart
// line 49-57: 初始化
if (current.id != _questionId || _exercises == null) {
  _exercises = List.from(_nextPracticeRound(
    _practiceExercises(current, practiceContext),
    questionId: current.id,
  ));
}

// line 539-551: 选择练习来源
List<GeneratedExercise> _practiceExercises(current, practiceContext) {
  final candidateId = practiceContext?.candidateId;
  if (candidateId == null) return current.savedExercises;  // 单题走这里
  // 多题拆分时从 candidateAnalyses 中找
  for (final candidate in current.candidateAnalyses) {
    if (candidate.candidateId == candidateId) {
      return candidate.savedExercises;
    }
  }
  return current.savedExercises;
}
```

### 从解析页进入练习 (`analysis_result_screen.dart:683-694`)

```dart
void _startPractice(record, activeCandidateAnalysis) {
  ref.read(currentPracticeContextProvider.notifier).state = PracticeContext(
    source: PracticeContextSource.analysis,
    candidateId: activeCandidateAnalysis?.candidateId,  // 单题时为 null
    returnRoute: '/analysis/result',
  );
  ref.read(currentQuestionProvider.notifier).state = record;
  context.go('/exercise/practice');
}
```

### 从详情页进入练习 (`question_detail_screen.dart:885-892`)

```dart
ref.read(currentPracticeContextProvider.notifier).state = PracticeContext(
  source: PracticeContextSource.notebook,
  returnRoute: '/notebook/question/${current.id}',
);
ref.read(currentQuestionProvider.notifier).state = current;
context.go('/exercise/practice');
```

### AI 解析完成时设置数据 (`analysis_loading_screen.dart:160-195`)

```dart
final generatedExercises = firstSuccessfulCandidate?.savedExercises ??
    service.extractGeneratedExercisesFromContent(...);

final updated = working.copyWith(
  savedExercises: generatedExercises,
  candidateAnalyses: candidateSnapshots.map(...).toList(),  // 单题时为空列表
);
ref.read(currentQuestionProvider.notifier).state = updated;
```

## 最可能的方向（未验证）

### 假设 A：多题拆分场景下 candidateId 不为 null

如果用户的题目触发了拆题（`splitResult?.hasMultipleCandidates == true`），那么：
- `activeCandidateAnalysis` 不为 null
- `practiceContext.candidateId` 被设置
- `_practiceExercises` 从 `candidateAnalyses[x].savedExercises` 取练习

需要验证：`candidateAnalyses[x].savedExercises` 中的 exercise 是否有 `diagramData`。

但这不能解释为什么"重新进入后有图"——因为重新进入时 `candidateId` 也会被设置（如果从详情页进入的话不会，因为 notebook 的 PracticeContext 没有 candidateId）。

### 假设 B：从详情页进入时 `current` 对象来源不同

- 场景 2（首次保存后立即练习）：notebook 列表从 `questionListProvider` 加载，该 provider 调用 `SharedPrefsQuestionRepository.listAll()`。如果 `invalidateQuestionList(ref)` 后 provider 还没重新加载完成，`current` 可能是旧的（没有 exercises 的）对象。
- 场景 3（重新进入）：provider 已经加载完成，`current` 是完整的。

### 假设 C：exercises 创建时 diagramData 实际为 null

`ai_analysis_service.dart` 中 `extractGeneratedExercisesFromContent` 解析 AI 返回的 JSON 时，`diagramData` 可能在某些情况下没被正确解析。需要检查 `ai_analysis_service.dart:1980` 附近的逻辑。

但这与"重新进入后有图"矛盾——如果创建时就没有，序列化后也不会有。

### 假设 D（最可能）：练习页的 `_questionId` 缓存导致不刷新

```dart
if (current.id != _questionId || _exercises == null) {
  _exercises = ...;
}
```

如果用户在同一个 session 中：
1. 先从解析页进入练习（此时 exercises 可能还没有 diagramData？或者有但被缓存了）
2. 返回后再进入，`current.id == _questionId` 为 true，不会重新加载 exercises

但这也不完全解释问题，因为 `_exercises` 是从 `current.savedExercises` 新建的。

## 建议下一步

1. **加 debugPrint 确认数据源**：在 `_practiceExercises` 方法中打印 `current.savedExercises.length` 和每个 exercise 的 `diagramData != null`

2. **确认是否多题拆分场景**：检查 `practiceContext?.candidateId` 是否为 null

3. **对比内存对象 vs 反序列化对象**：在 `_startPractice` 调用前打印 `record.savedExercises[0].diagramData`，确认内存中的对象是否真的有 diagramData

4. **检查 `extractGeneratedExercisesFromContent`**：看 `ai_analysis_service.dart:1970-2000` 附近，确认 AI 返回的 exercises 是否正确解析了 diagramData

## 关键文件清单

| 文件 | 作用 |
|------|------|
| `lib/src/features/analysis/presentation/exercise_practice_screen.dart` | 练习页面，bug 表现位置 |
| `lib/src/features/analysis/presentation/analysis_result_screen.dart` | 解析结果页，发起练习入口 |
| `lib/src/features/analysis/presentation/analysis_loading_screen.dart` | AI 解析流程，生成 exercises |
| `lib/src/features/notebook/presentation/question_detail_screen.dart` | 错题详情页，另一个练习入口 |
| `lib/src/app/providers.dart` | currentQuestionProvider 定义 |
| `lib/src/domain/models/generated_exercise.dart` | Exercise 模型 + copyWith |
| `lib/src/domain/models/question_record.dart` | QuestionRecord 模型 |
| `lib/src/data/repositories/shared_prefs_question_repository.dart` | 实际使用的持久化层 |
| `lib/src/data/remote/ai/ai_analysis_service.dart:1970-2030` | AI 返回 exercises 的解析逻辑 |
