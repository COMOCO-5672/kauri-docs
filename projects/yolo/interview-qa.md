# YOLO 面试问答

> [!NOTE] 版本快照
> ultralytics `8.4.115` · torch `2.13.0` · Python `3.12` · 文档日期 `2026-08-02`。ultralytics 为 pip 安装包（非 git 仓库），无 commit/describe，以包版本号为准。

这些题目用来检验对 ultralytics YOLO 的架构理解、源码定位能力和工程判断。适合作为 CV/AI 转型面试的自测。

## 架构理解

**Q1: 用一行话说清 YOLO 检测模型输出什么？**
模型对每个预设锚点/尺度位置输出框（坐标）+ 类别分数。推理时经 DFL 解码框、sigmoid 分类、NMS 去重后，最终给出一组 `[x1,y1,x2,y2, conf, class_id]` 的检测结果。输出形态见 `models/yolo/detect/predict.py:construct_result()`。

**Q2: 训练时 Detect head 输出和推理时有什么不同？为什么？**
训练时 `Detect.forward()` 返回原始 dict（`{"boxes","scores","feats"}`），交给 `v8DetectionLoss` 算 loss；推理时走 `_inference()` 解码成框 + 分数。因为 loss 需要的是"未经过 NMS/解码的原始预测"，而用户要的是"可用的框"。见 `nn/modules/head.py:Detect.forward()` / `_inference()`。

**Q3: `YOLO` 类是怎么派发到具体任务实现（detect/segment/pose）的？**
`YOLO.task_map` 维护任务 → 类的映射。`train/val/predict` 通过 `_smart_load("trainer"/"validator"/"predictor")` 从映射里取对应子类（如 `DetectionTrainer`）。见 `models/yolo/model.py:task_map`、`models/yolo/detect/__init__.py`。

**Q4: `parse_model()` 做了什么？为什么模型用 yaml 描述？**
`parse_model()` 把 yaml 里每行 `[from, number, module, args]` 解析成 `torch.nn.Sequential` 的具体模块，并推导通道数、记录需要保存的中间层索引。用 yaml 描述让结构可配置、可 diff、不用改 Python 代码就能改网络结构。见 `nn/tasks.py:parse_model()`。

## 数据与训练

**Q5: 为什么训练 loss 下降但验证结果差？可能的原因有哪些？**
过拟合（数据少、增强弱）、训练/验证分布不一致、数据标注错误、超参不当。重点：**数据问题不报错但会让模型学错**，验证前先可视化标注。见 `data/augment.py` 的增强管线。

**Q6: 标签文件 `cls xc yc w h` 中坐标为什么是归一化的？**
归一化让坐标与图像分辨率解耦，同一标注在不同分辨率下通用，且数值稳定利于训练。如果存像素坐标，换分辨率就得重标。见 `data/base.py` 的标签处理。

**Q7: `v8DetectionLoss` 的三部分 loss 是什么？TAL 起什么作用？**
box（BboxLoss）、cls（BCE）、dfl（DFLoss 分布）。`TaskAlignedAssigner`（TAL）先做标签分配——决定每个预测框匹配哪个真值框——再计算三路 loss。见 `utils/loss.py:v8DetectionLoss`、`utils/tal.py:TaskAlignedAssigner`。

**Q8: 为什么 `loss * batch_size`？**
`v8DetectionLoss.loss()` 返回的是整个 batch 的总 loss，不是单样本平均。对比不同 batch 的 loss 曲线要归一化。见 `utils/loss.py`。

## 推理与部署

**Q9: letterbox 之后为什么框会"偏了"？怎么解决？**
letterbox 缩放并填充到模型输入尺寸，网络输出的是填充后画布坐标。必须用 `ops.scale_boxes` 映射回原图尺寸。跳过这步就会看到框偏移。见 `models/yolo/detect/predict.py:construct_result()`。

**Q10: `model.predict(stream=True)` 和普通 `predict` 有什么区别？**
普通 `predict` 一次性返回 `list[Results]`；`stream=True` 返回迭代器，逐帧产出 `Results`，适合视频/摄像头，内存友好。见 `engine/predictor.py:BasePredictor.stream_inference()`。

**Q11: 导出 TensorRT 失败，可能的原因有哪些？**
不一定是代码问题：需要 NVIDIA GPU、CUDA Toolkit、TensorRT 环境。先查运行时依赖再改代码。"支持导出"≠"环境能导出"。见 `engine/exporter.py:Exporter`。

**Q12: 检测结果为空，怎么区分是"没检测到"还是"阈值过滤了"？**
用低阈值 `predict(conf=0.001, iou=0.9)` 看有没有低置信度输出。有 = 阈值过滤；没有 = 模型没检测到。见 `utils/nms.py:non_max_suppression`。

## 工程判断

**Q13: 实时摄像头检测，先看哪个指标定位性能瓶颈？**
`Results.speed` 分 `preprocess/inference/postprocess` 三段耗时。先定位哪段占比高，再针对性优化（换后端、降分辨率、换小模型）。见 `engine/results.py:Results.speed`。

**Q14: 客户要识别"自家产线上的螺丝"，你会怎么规划？**
明确这是自定义类别，必须：收集数据 → 标注（YOLO 格式）→ 训练/微调 → 验证 → 部署。第一步先可视化确认标注正确，用 `coco8` 冒烟验证环境，再上真实数据。预训练 COCO 模型不认螺丝，这步绕不过去。

**Q15: 为什么说"跟踪质量上限是检测质量"？**
跟踪器（ByteTrack 等）靠检测框做跨帧关联，检测漏检/错检会直接污染跟踪结果。检测不稳时调跟踪参数没用，要先提升检测。见 `trackers/`。

**Q16: 你要在一台 M4 Pro 48GB 上部署实时检测，怎么做决策？**
先看内存（决定模型/batch 能多大），再看算力（决定吞吐）。M4 Pro 上 PyTorch+MPS 推理 YOLOv8n 已很实时；若追求极致或要上 iOS，再考虑 CoreML 导出。判断依据：`Results.speed` + 实际帧率测试，不靠猜。
