// mobile_app/lib/models/enhancer_model.dart (Дополнение)

// ... (существующие enum EnhancementType, AIProcessingStatus - остаются прежними) ...

// Представление отдельного шага в истории редактирования
class HistoryStep {
  final String stepId;
  final String description; // Например, "Применено шумоподавление AI 0.7"
  final EnhancementConfig? config; // Конфигурация, которая была применена
  final DateTime timestamp;

  HistoryStep({required this.stepId, required this.description, this.config})
      : timestamp = DateTime.now();
}

// Продвинутая модель задачи обработки (Job)
class ProcessingJob {
  final String jobId;
  final String originalAssetId;
  final List<HistoryStep> appliedSteps; // Ссылка на шаги, которые были применены
  AIProcessingStatus status;
  String? resultImageUrl;
  String? errorMessage;
  
  ProcessingJob({
    required this.jobId, required this.originalAssetId, required this.appliedSteps, 
    this.status = AIProcessingStatus.pending, this.resultImageUrl, this.errorMessage
  });
}
