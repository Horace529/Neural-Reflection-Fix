
## 一、项目概述

SSR-Guardian 是一种面向入门级与中端光线追踪 GPU 的实时混合反射增强方案，目标是在有限的硬件光追预算下，提升屏幕空间反射（Screen-Space Reflection, SSR）在动态场景中的稳定性与视觉质量。

传统 SSR 计算开销低、易于实时实现，但其反射可见性信息受限于屏幕空间，在动态场景、物体边缘、遮挡变化及屏幕空间信息缺失的区域，容易出现断裂、漏反射、错误反射与闪烁等伪影。另一方面，全屏硬件光线追踪虽然能给出更可靠的反射结果，但计算成本高，中低端 GPU 难以在全程开启的情况下维持稳定帧率。

已有 Hybrid Reflection 类工作表明，先由分类器识别需要光追修正的区域，再对这些区域执行局部光线追踪，可以在 SSR 与全屏硬件光追之间取得性能与画质的折中。因此，本项目不再将"SSR 错误检测 + 局部 RT 修复"本身作为创新点，而是进一步研究：

> **能否利用轻量级神经网络，在固定的光追计算预算下，预测各反射区域经硬件光追后所能获得的视觉收益，从而把有限的 RT 资源优先分配给最值得修复的区域？**

基于这一思想，SSR-Guardian 将神经网络定位为一个轻量级的"智能光追调度器"，而非图像生成器。网络不直接预测最终颜色，也不试图替代传统渲染管线，而是综合当前帧与历史帧中的几何、材质、SSR 及运动信息，预测各反射区域的错误风险与 RT 修复收益；随后由 GPU 端调度器在预设的 Ray Budget 内，挑选最值得执行硬件光追的区域。

最终形成的分工如下：

**传统光栅化 / SSR 低成本地覆盖大部分区域；神经网络负责识别高价值修复区域；硬件 RT 负责为这些区域提供物理上更可靠的反射结果。**

这一设计旨在为有限算力的 GPU 提供一种"按需计算"的反射渲染框架。

---

## 二、研究目标

本项目围绕以下四个问题展开：

1. **SSR 伪影能否被轻量级神经网络稳定预测？**

   研究利用深度、法线、材质、SSR 结果以及运动与时序信息，对 SSR 潜在错误区域进行分类或收益预测的可行性。

2. **在相同 RT 计算预算下，神经调度能否优于传统启发式分类方法？**

   不以"尽可能减少 RT 像素"为唯一目标，而是研究如何把有限的 RT budget 分配给视觉收益最高的区域。

3. **如何抑制动态场景中的时序闪烁与分类不稳定？**

   将运动向量、历史帧置信度及历史反射信息引入预测过程，使模型不仅能利用单帧空间特征，还能借助时域信息判断 SSR 错误是否具有持续性。

4. **如何构建更符合实际感知的 SSR 错误监督信号？**

   不再只比较 SSR 与参考图像的像素颜色差异，而是结合反射命中几何、可见性与感知误差构造训练标签，让网络学习"值得用 RT 修复的错误"，而不是简单地学习"两幅图像颜色不同的地方"。

---

## 三、核心技术路线

### 1. 数据生成与监督信号构建

项目首先搭建离线数据生成管线，在不同材质、光照、摄像机运动与物体运动条件下生成大量训练样本。

每帧数据主要包括：

- Depth
- Normal
- Albedo
- Roughness
- Metallic / 材质信息
- Motion Vector
- SSR 结果
- 上一帧重投影信息
- 高质量硬件光线追踪 / 路径追踪生成的参考反射

训练标签不再以"SSR 与参考图像 RGB 差异是否超过固定阈值"作为唯一标准，而是综合构建几何感知与视觉感知的 SSR error / benefit map，具体涵盖三个维度：

**Appearance Error：**

衡量 SSR 与高质量参考结果在 HDR luminance、颜色及感知空间上的差异。

**Geometric / Visibility Error：**

借助参考光线追踪结果中的 hit position、hit primitive / object ID、visibility 等信息，判断 SSR 是否发生了错误的反射事件。

**Temporal Error：**

通过当前帧与历史帧的重投影一致性，判断错误是否具有明显的时间不稳定性。

在此基础上生成三类监督信号：

- SSR Error Mask
- SSR Error Confidence
- Expected RT Benefit

其中 Expected RT Benefit 是后续"预算分配"研究的核心目标。

---

### 2. Tile-Level Neural Guardian

考虑到现代 GPU 的执行与光线追踪调度特点，本项目不以单个像素作为调度单位，而采用 8×8 或 16×16 的 tile 作为基本决策单位。

输入数据经适当降采样后，送入轻量级卷积神经网络或其他低成本视觉网络。输入信息包括：

- 低分辨率 G-Buffer
- SSR radiance / color
- Motion Vector
- Temporal confidence
- 其他反射相关辅助特征

网络为每个 tile 输出：

$$
P(\text{error})
$$

以及：

$$
P(\text{benefit}\mid \text{RT})
$$

或者直接预测一个综合的 RT Priority Score：

$$
S_i = \frac{\Delta Q_i}{C_i}
$$

其中：

- $\Delta Q_i$：该 tile 使用 RT 后预计获得的视觉质量提升；
- $C_i$：该区域对应的计算成本。

因此，网络的作用并不是"重新画出正确的反射"，而是回答：

> **哪些地方值得消耗有限的 RT 计算资源？**

这也是 SSR-Guardian 与直接神经图像重建方法的主要区别。

---

### 3. Temporal Neural Prediction

针对动态场景中 SSR 容易闪烁、不稳定的问题，引入历史帧信息：通过 Motion Vector 将上一帧的相关特征重投影至当前帧，并结合上一帧置信度、重投影有效性、历史 SSR 结果与当前帧几何进行联合判断，从而缓解摄像机轻微运动、物体运动及局部信息变化引起的分类抖动。

同时，可以设计简单的 temporal consistency loss，使连续帧中对同一反射事件的预测保持稳定。

---

### 4. Budget-Constrained RT Scheduling

这是项目的核心研究环节。与"置信度超过固定阈值即执行 RT"的简单策略不同，SSR-Guardian 采用固定或动态的 Ray Budget，例如对同一场景分别设置 5%、10%、15%、20% 的 RT budget。

网络先计算各 tile 的 RT Priority Score，再由 GPU 端调度器按得分从高到低选择 tile。系统的优化目标可表示为：

$$
\max \sum_i \Delta Q_i x_i
$$

subject to：

$$
\sum_i C_i x_i \leq B
$$

其中：

- $\Delta Q_i$：第 $i$ 个 tile 使用 RT 后的预期视觉收益；
- $C_i$：该 tile 的 RT 计算成本；
- $x_i$：是否执行 RT；
- $B$：本帧允许使用的 RT 总预算。

这一机制使系统从简单的"错误检测器"进一步转化为**面向有限计算资源的神经引导 Ray Allocation 系统**。

---

### 5. GPU Ray Tracing 调度与结果合成

在 DX12 / DXR 环境中，神经网络输出的 tile priority 将被转化为 GPU 可执行的 ray list 或间接调度任务。整体流程为：

**SSR → Neural Guardian → Tile Classification → GPU Compaction → DXR → Result Composite**

对被选中的区域执行硬件光线追踪反射计算，并将 RT 结果覆盖或融合进 SSR 结果；未进入 RT budget 的区域则继续使用 SSR 或其他低成本反射结果。在条件允许时，可进一步借助 DXR 1.1 的 GPU-driven / indirect dispatch 机制减少 CPU 参与，实现完整的 GPU-side scheduling。

---

## 四、网络设计

网络定位为面向实时推理的轻量级特征分类器，而非图像生成网络。结构上计划采用轻量级 CNN、Depthwise Separable Convolution 等便于部署的形式，目标包括：

- 参数量控制在数百万级；
- FP16 推理；
- 低显存占用；
- 低 latency；
- 输出 tile-level prediction；
- 尽量避免干扰主渲染线程。

相比此前将模型规模设定为 50–100 MB 的设想，本项目更强调模型轻量化与实际 GPU kernel overhead，力求将模型规模控制在实时渲染管线能够轻松容纳的范围内。

部署环境采用 NVIDIA TensorRT，并针对 RTX GPU 与 FP16、INT8 等精度开展实验；TensorRT 对实时、低延迟推理场景已有充分优化。

---

## 五、系统整体架构

SSR-Guardian 的完整渲染流程如下：

```text
                    ┌──────────────────────┐
                    │   Current G-Buffer   │
                    │ Depth / Normal / etc │
                    └──────────┬───────────┘
                               │
                               │
                 ┌─────────────▼─────────────┐
                 │       SSR Rendering       │
                 └─────────────┬─────────────┘
                               │
                               │
        ┌──────────────────────▼──────────────────────┐
        │           Temporal Neural Guardian          │
        │                                             │
        │ G-Buffer + SSR + Motion + History Features  │
        └──────────────────────┬──────────────────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │ Tile Error / RT Benefit│
                  │      Prediction        │
                  └────────────┬───────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │ Budget-Constrained     │
                  │ Ray Allocation         │
                  └────────────┬───────────┘
                               │
                               ▼
                     GPU Ray Compaction
                               │
                               ▼
                         DXR / RT Core
                               │
                               ▼
                     RT Reflection Result
                               │
                               ▼
                 ┌────────────────────────┐
                 │   SSR + Selective RT   │
                 │       Composite        │
                 └────────────┬───────────┘
                              │
                              ▼
                         Final Image
```

---

## 六、项目创新点

本项目不以"SSR + 局部硬件 RT"本身作为创新点，而聚焦于以下三个方面。

### 创新点一：面向 RT Budget 的神经引导反射调度

将神经网络的任务从"生成最终图像"转变为"预测 RT 投资收益"，在固定光追预算下选择最值得计算的区域。与直接的神经图像修复相比，该方案保留了传统渲染管线的确定性，同时利用硬件 RT 获取高可靠的反射结果。

---

### 创新点二：面向动态场景的时序神经分类

将当前帧的几何与 SSR 特征同上一帧信息结合，利用运动向量和 temporal confidence 预测 SSR artifact，提升动态场景下分类结果的稳定性。重点研究：

> **Temporal consistency 是否能在不明显增加网络计算量的前提下，提高低 RT budget 下的反射稳定性。**

---

### 创新点三：面向视觉收益的几何感知监督

传统 SSR error detection 容易把单纯的颜色差异等同于视觉错误。本项目进一步引入反射命中信息、几何一致性、可见性信息、感知外观差异与时序一致性，构建更接近真实渲染质量评价的监督信号。网络的学习目标由此从：

> "SSR 与参考图像哪里不同"

转变为：

> "哪里存在值得消耗 RT 资源修复的反射错误"。

---

## 七、与已有方法的区别

已有 Hybrid Reflection 工作已经证明，用分类器把 SSR 与光线追踪结合是一条有效的工程路线，例如 AMD FidelityFX Hybrid Reflections 便采用反射分类与选择性光线追踪来控制反射计算成本。因此，本项目的重点不是重复证明"SSR + 局部 RT 可行"，而是研究：

**传统启发式分类器 → 学习型分类器 → 面向 RT Budget 的神经收益预测**

即在相同 RT budget 下，学习型调度器能否比传统 heuristic classifier 获得更好的反射质量。项目还将与全屏 Ray Tracing、纯 SSR 以及其他 Hybrid Reflection 方法进行公平比较。

---

## 八、实验设计

为验证方法的有效性，计划设置以下主要 baseline：

| 方法 | RT Budget | 主要目的 |
|---|---:|---|
| Pure SSR | 0% | 基准画质与性能 |
| Full Ray Tracing | 100% | 高质量参考 |
| Naive Hybrid | 固定比例 | 基础混合方案 |
| Heuristic Hybrid | 5–20% | 与传统分类方法比较 |
| SSR-Guardian | 5–20% | 验证神经调度效果 |
| Temporal SSR-Guardian | 5–20% | 验证时序信息贡献 |

评价指标如下。

### 性能指标

- Average FPS
- Frame Time
- AI inference time
- Scheduling / Compaction time
- DXR time
- GPU memory consumption

### 图像质量指标

- PSNR
- SSIM
- LPIPS
- HDR / perceptual difference
- SSR artifact rate

### 核心研究指标

重点定义：

$$
ASR = \frac{E_{SSR}-E_{Guardian}}{E_{SSR}-E_{RT}}
$$

其中 $E$ 为误差指标，数值越低代表质量越好。ASR 用于衡量在有限 RT budget 下，SSR-Guardian 消除了多少 SSR 与高质量 RT 之间的质量差距：ASR 为 1 表示修复效果达到全屏 RT 水平，为 0 表示与纯 SSR 相当。

同时绘制：

> **Visual Quality vs. RT Budget**

曲线，用于验证在相同光追成本下，神经调度是否优于传统调度方法。

---

## 九、预期成果

项目计划最终形成一个完整的实时渲染原型系统，并围绕以下研究问题撰写论文：

> **在固定硬件光追预算下，时序神经模型是否能够比传统启发式方法更有效地选择需要 RT 修复的反射区域？**

预期成果包括：

1. 一个基于 DX12 / DXR 的实时 Hybrid Reflection 原型；
2. 一个能够进行 SSR artifact / RT benefit prediction 的轻量级神经网络；
3. 一个面向固定 Ray Budget 的 GPU-side adaptive scheduling pipeline；
4. 一套包含动态场景、不同材质与不同运动条件的训练及测试数据集；
5. 对比 Pure SSR、Full RT、Heuristic Hybrid 与 Neural Hybrid 的完整实验结果；
6. 一篇围绕"有限光追预算下的神经引导自适应反射渲染"的论文。

---

## 十、预期性能目标

项目初期不对最终性能作过度承诺，而是采用逐步验证的指标体系。

第一阶段目标：

- 1080p 实时运行；
- 神经网络推理保持亚毫秒级；
- RT budget 控制在约 5%–20%；
- 相比纯 SSR 明显降低主要反射伪影；
- 相比相同 RT budget 的 heuristic hybrid 方法获得更高图像质量。

第二阶段进一步研究：

- 降低 neural inference 与 scheduling overhead；
- 提升 GPU-side ray compaction 效率；
- 研究 5%、10%、15%、20% 不同 Ray Budget 下的质量—性能曲线；
- 评估 Temporal Neural Guardian 对动态场景 flickering 的改善程度。

最终不以"接近全屏 RT"作为唯一目标，而以：

> **在有限 RT budget 下最大化单位计算成本所获得的视觉质量提升**

作为项目的核心评价标准。

---

## 十一、项目最终定位

SSR-Guardian 既不尝试用 AI 重新实现传统渲染，也不让神经网络直接生成反射图像，而是解决一个更具体的问题：

> **当 GPU 没有足够的计算资源让所有像素都执行高质量 Ray Tracing 时，如何让系统自动决定"哪里最值得计算"。**

传统 SSR 负责低成本覆盖大部分区域；硬件 Ray Tracing 负责高价值、高风险区域；轻量级神经网络则连接两者，依据场景、时序与视觉信息动态分配有限的计算资源。

因此，SSR-Guardian 的核心思想可以概括为：

> **不是让 AI 替代渲染，而是让 AI 决定哪些地方值得渲染得更精确。**
