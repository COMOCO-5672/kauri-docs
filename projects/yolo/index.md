# Ultralytics YOLO Project Framework

源码快照：

- 库名：`ultralytics`（Ultralytics YOLO）
- 版本：`8.4.115`
- 安装方式：venv pip 安装（`pip install ultralytics`）
- 文档日期：2026-08-02

## 文档

- [项目概览：能做什么](what-yolo-can-do.md)
- [整体架构：模型、引擎与数据流](architecture.md)
- [训练数据格式与标注管线](training-data-format.md)
- [推理与部署：从 Python 到生产](inference-and-deployment.md)
- [平台与运行环境矩阵](platform-and-runtime-matrix.md)
- [工程问题手册](engineering-playbook.md)
- [面试问答](interview-qa.md)

## 图示索引

- ultralytics 分层架构图：见 `architecture.md`。
- 训练主循环图：见 `architecture.md`。
- 推理前处理到后处理流程图：见 `architecture.md`。
- 数据集目录格式图：见 `training-data-format.md`。
- 数据增强管线图：见 `training-data-format.md`。
- 导出目标格式决策图：见 `inference-and-deployment.md`。

## 快速定位

- 用户入口：`engine/model.py:Model`，`models/yolo/model.py:YOLO`。
- 训练：`engine/trainer.py:BaseTrainer`，`models/yolo/detect/train.py:DetectionTrainer`。
- 推理：`engine/predictor.py:BasePredictor`，`models/yolo/detect/predict.py:DetectionPredictor`。
- 验证：`engine/validator.py:BaseValidator`，`models/yolo/detect/val.py:DetectionValidator`。
- 模型定义：`nn/tasks.py:BaseModel` / `DetectionModel` / `parse_model()`。
- 网络模块：`nn/modules/conv.py`（`Conv`）、`nn/modules/block.py`（`C2f`、`SPPF`、`DFL`）、`nn/modules/head.py`（`Detect`）。
- Loss：`utils/loss.py:v8DetectionLoss`，标签分配 `utils/tal.py:TaskAlignedAssigner`。
- NMS：`utils/nms.py:non_max_suppression`。
- 结果对象：`engine/results.py:Results`（`Boxes`/`Masks`/`Keypoints`）。
- 配置：`cfg/default.yaml`，模型 `cfg/models/v8/yolov8.yaml`。
