// mobile_app/lib/providers/active_editor_provider.dart (Обновление)
// ... (импорты) ...
import '../services/local_preview_service.dart';
import '../services/job_queue_manager.dart';

// Новое состояние для истории
class HistoryStack {
  final List<HistoryStep> stack;
  final int currentIndex; // Указывает на последний активный шаг

  HistoryStack({this.stack = const [], this.currentIndex = -1});
  
  // Добавление нового шага
  HistoryStack push(HistoryStep step) {
    // Обрезаем "отмененную" часть истории
    final newStack = stack.sublist(0, currentIndex + 1);
    newStack.add(step);
    return HistoryStack(stack: newStack, currentIndex: newStack.length - 1);
  }

  // Отмена (Undo)
  HistoryStack undo() {
    if (currentIndex > -1) {
      return HistoryStack(stack: stack, currentIndex: currentIndex - 1);
    }
    return this;
  }
  
  // Повтор (Redo)
  HistoryStack redo() {
    if (currentIndex < stack.length - 1) {
      return HistoryStack(stack: stack, currentIndex: currentIndex + 1);
    }
    return this;
  }
}

class ActiveEditorState {
  // ... (предыдущие поля) ...
  final HistoryStack history; // Новый стек истории

  ActiveEditorState({
    // ... (предыдущие поля) ...
    this.history = const HistoryStack(),
  });

  ActiveEditorState copyWith({
    // ... (предыдущие поля) ...
    HistoryStack? history,
  }) {
    return ActiveEditorState(
      // ... (предыдущие поля) ...
      history: history ?? this.history,
    );
  }
}

// ... (activeEditorProvider и ActiveEditorNotifier - остаются) ...

class ActiveEditorNotifier extends StateNotifier<ActiveEditorState> {
  // ... (предыдущие поля) ...

  // --- Новая функция: Применение одного шага редактирования ---
  Future<void> applySingleStep(EnhancementConfig config, {bool isAIStep = true}) async {
    if (state.originalPhoto == null) return;
    
    final step = HistoryStep(
      stepId: DateTime.now().millisecondsSinceEpoch.toString(),
      description: config.type.name,
      config: config,
    );
    
    // 1. Обновляем историю
    state = state.copyWith(history: state.history.push(step));
    
    // 2. Локальный предпросмотр (для немедленного фидбека)
    final localPreviewPath = await _ref.read(localPreviewServiceProvider).applyLocalAdjustment(
      File(state.originalPhoto!.path), 
      config,
    );
    state = state.copyWith(currentPreviewPath: localPreviewPath);
    
    // 3. Отправляем на бэкенд, если это ИИ-эффект
    if (isAIStep) {
        _submitJobFromHistory();
    }
  }
  
  // --- Отмена и Повтор (Undo/Redo) ---
  void undo() {
    state = state.copyWith(history: state.history.undo());
    // После Undo/Redo необходимо пересчитать финальный вид изображения.
    // В реальном приложении: перерендеринг или загрузка кешированного состояния
    _triggerFullRender();
  }
  
  void redo() {
    state = state.copyWith(history: state.history.redo());
    _triggerFullRender();
  }
  
  // --- Создание финальной задачи из активной истории ---
  void _submitJobFromHistory() {
    if (state.originalPhoto == null) return;

    // Берем только активные шаги из истории
    final activeSteps = state.history.stack.sublist(0, state.history.currentIndex + 1);

    // Создаем новую задачу с текущим стеком шагов
    final job = ProcessingJob(
      jobId: 'JOB-${DateTime.now().millisecondsSinceEpoch}',
      originalAssetId: state.originalPhoto!.id,
      appliedSteps: activeSteps,
    );

    // Отправляем задачу в менеджер очереди
    _ref.read(jobQueueManagerProvider).submitJob(job);
    state = state.copyWith(activeJob: job); // Устанавливаем эту задачу как активную
  }
  
  // Имитация: Запуск рендеринга на основе истории
  void _triggerFullRender() {
    print('RENDER: Запуск полного рендеринга на основе ${state.history.currentIndex + 1} шагов истории.');
    // Здесь должна быть логика: 
    // 1. Проверка, есть ли уже готовый результат (resultImageUrl) для этого стека шагов.
    // 2. Если нет, отправить _submitJobFromHistory() и ждать результат.
  }

  // ... (dispose - остается) ...
}
