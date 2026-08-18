# Fun3DU

## 整体框架

Fun3DU 的目标是：给定一个自然语言任务，在 3D 场景中找到真正需要被操作的功能部件，并输出它的 3D segmentation mask。

整体流程可以概括为：

\[ \text{Task Description} \rightarrow \text{Task Understanding} \rightarrow \text{Contextual Object Grounding} \rightarrow \text{View Selection} \rightarrow \text{Functional Object Grounding} \rightarrow \text{2D-to-3D Aggregation} \]

其中，论文区分了两个关键概念：

- **Functional Object \(F\)**：真正需要被操作的功能部件，例如 handle、switch、knob。
- **Contextual Object \(O\)**：帮助确定 Functional Object 所在位置的上下文对象，例如 cabinet、door、lamp。

例如任务：

> Open the bottom drawer of the cabinet with the TV on top.

模型需要推理出：

\[ F=\text{handle}, \qquad O=\text{cabinet} \]

之后先找到目标 cabinet，再在与 cabinet 相关的视角中寻找真正需要操作的 handle，最后将多视角的 2D 结果融合回 3D 点云。

## Stage 1：Task Description Understanding

第一阶段使用 LLM 理解任务描述 \(D\)，并推理出 Functional Object \(F\) 和 Contextual Object \(O\)：

\[ D\xrightarrow{\text{LLM}}(F,O) \]

这里的关键是，任务描述中往往不会直接出现 Functional Object。例如：

> Turn on the ceiling light.

真正需要操作的是：

\[ \text{ceiling light}\rightarrow\text{light switch} \]

因此 Fun3DU 不只是做文本类别到物体的 grounding，而是先完成一次 task-level functional reasoning。

同时，LLM 还需要确定合适的功能层级。例如对于 “open drawer”，最终目标应该是 handle，而不是继续推理到 hand 或 finger。论文通过机器人能够执行的操作集合约束 LLM 的推理范围。

## Stage 2：Contextual Object Grounding 与 View Selection

得到 Contextual Object \(O\) 后，Fun3DU 使用 OWLv2 和 RobustSAM 在大量 RGB-D views 中定位该对象。

例如：

\[ O=\text{cabinet} \]

此时模型先寻找 cabinet，而不是直接在整个场景中搜索 handle。这样可以把：

\[ \text{scene-level search} \rightarrow \text{context-conditioned local search} \]

SceneFun3D 中一个场景可能包含约 1800 张 RGB-D 图像，因此作者进一步进行 View Selection。每个 view 的评分综合考虑 Contextual Object 的检测置信度，以及其 mask 在图像中的位置和分布：

\[ S_O^n= \lambda_mS_{m_O}^n+ \lambda_dS_{d_O}^n+ \lambda_\alpha S_{\alpha_O}^n \]

最终选择大约 50 个最适合观察 Contextual Object 的视角。

这一阶段的作用不只是降低计算量，更重要的是缩小后续 VLM 的搜索空间：原本是在整个房间中找 handle，现在变成在“目标 cabinet 附近”找 handle。

## Stage 3：Functional Object Grounding

在筛选后的高质量视角中，Fun3DU 使用 Molmo 定位 Functional Object。

输入不仅包含 Functional Object \(F\)，还保留完整的 Task Description \(D\)：

\[ F+D+\text{Image}\rightarrow\text{2D Point} \]

例如：

> Point to all the handles in order to open the bottom drawer of the cabinet with the TV on top.

保留完整任务描述非常重要，因为仅仅知道 `handle` 无法区分 door handle、drawer handle 或其他 handle。任务中的 bottom drawer、cabinet、TV on top 等描述实际上提供了额外的空间和语义关系。

Molmo 输出目标 point，随后使用 SAM 将 point 扩展为精确的 2D mask：

\[ \text{Molmo Point}\xrightarrow{\text{SAM}}\text{2D Functional Mask} \]

因此这里可以理解为：

\[ \text{VLM负责语义定位}+\text{SAM负责精确分割} \]

## Stage 4：Multi-view 2D-to-3D Aggregation

得到多个视角中的 Functional Object mask 后，Fun3DU 利用 RGB-D、相机内外参和 3D point cloud，将 2D mask 投影回 3D：

\[ \Gamma^k:\text{2D Pixel}\rightarrow\text{3D Point} \]

由于单个 view 的 VLM 或 SAM 可能产生误检，论文没有简单地将所有 2D mask 做 union，而是使用 **Multi-view Agreement**。

对于每个 3D 点 \(c_i\)，统计它在多少个不同视角中被预测为 Functional Object：

\[ s_i= \sum_{k=1}^{K} \left| \left\{ p^k:\Gamma^k(p^k)=c_i,\;p^k\in m^k \right\} \right| \]

如果一个 3D 点是真正的 handle，它通常会在多个视角中反复被识别；偶然误检则只会出现在少数视角。经过归一化和阈值筛选后得到最终的 3D mask：

\[ \mathcal M=\{c_i\mid s_i>\tau\} \]

## 总结

Fun3DU 可以简化成四步：

\[ \boxed{ \text{Task} \xrightarrow{\text{LLM}} (F,O) \xrightarrow{\text{Context Grounding}} \text{Relevant Views} \xrightarrow{\text{VLM+SAM}} \text{2D Functional Masks} \xrightarrow{\text{Multi-view Fusion}} \text{3D Functional Mask} } \]

它最重要的特点是把传统的：

\[ \text{Text Category}\rightarrow\text{3D Object} \]

扩展成：

\[ \text{Task}\rightarrow\text{Functional Part}\rightarrow\text{3D Functional Region} \]

因此，Fun3DU 更适合被理解为 **Task-conditioned 3D Functional Part Grounding**：它回答的是“为了完成这个任务，机器人应该操作哪个部件，以及这个部件在 3D 场景中的哪里”。