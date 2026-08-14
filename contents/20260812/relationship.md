relationship 可以分为：

- 空间关系（left of / right of / above / below）
- 结构关系（part-of / connected-to）
- Semantic / functional relationship

空间关系可以通过几何方法直接计算。

结构关系需要识别**类别可泛化**的 actionable parts（[GAPartNet](https://arxiv.org/abs/2211.05272)）。

semantic / functional relationship 一般是通过：

\[\text{3D geometry + multi-view RGB + VLM / LLM knowledge}\]

[ConceptGraphs](https://arxiv.org/abs/2309.16650)

[Open-Vocabulary Functional 3D Scene Graphs for Real-World Indoor Spaces](https://arxiv.org/abs/2503.19199)
