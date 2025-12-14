// mobile_app/lib/services/job_queue_manager.dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/enhancer_model.dart';
import 'image_processing_api.dart';

final jobQueueManagerProvider = Provider((ref) => JobQueueManager(ref.read(imageProcessingApiProvider)));

class JobQueueManager {
  final ImageProcessingApi _api;
  final List<ProcessingJob> _jobQueue = [];
  final StreamController<List<ProcessingJob>> _queueStreamController = StreamController.broadcast();
  Stream<List<ProcessingJob>> get queueStream => _queueStreamController.stream;
  Timer? _pollingTimer;

  JobQueueManager(this._api) {
    _startPolling();
  }

  // Правило JOB-001: Добавление задачи в очередь
  void submitJob(ProcessingJob job) {
    _jobQueue.add(job);
    _queueStreamController.add([..._jobQueue]);
    print('JOB QUEUE: Добавлена новая задача ${job.jobId}. Текущая очередь: ${_jobQueue.length}');
    
    // В реальном приложении: Отправка на бэкенд
    // _api.submitEnhancementJob(photo, configs).then(...)
  }

  // Правило JOB-002: Периодический опрос статуса активных задач
  void _startPolling() {
    _pollingTimer = Timer.periodic(const Duration(seconds: 5), (timer) async {
      final activeJobs = _jobQueue.where((j) => j.status == AIProcessingStatus.pending || j.status == AIProcessingStatus.inProgress).toList();
      
      for (var job in activeJobs) {
        try {
          final updatedJob = await _api.getJobStatus(job.jobId);
          
          if (updatedJob.status == AIProcessingStatus.completed || updatedJob.status == AIProcessingStatus.failed) {
            // Задача завершена, удаляем ее из активного списка
            job.status = updatedJob.status;
            job.resultImageUrl = updatedJob.resultImageUrl;
            
            print('JOB QUEUE: Задача ${job.jobId} завершена. Статус: ${job.status.name}');
            // Здесь может быть вызов колбэка для обновления UI
          }
        } catch (e) {
          print('JOB QUEUE ERROR: Не удалось получить статус для ${job.jobId}: $e');
        }
      }
      _queueStreamController.add([..._jobQueue]);
    });
  }

  void dispose() {
    _pollingTimer?.cancel();
    _queueStreamController.close();
  }
}
