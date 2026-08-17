# FunFact 中 Functional Edge 是怎么决定的？

可以把 FunFact 的过程理解成两步：

1. 先把“可能存在的边”全部列出来；
2. 不直接决定某一条边，而是先比较“整个连接方案”有多合理，再反过来计算每条边存在的概率。

最后需要做一个离散决定：

$$
P(x_i = 1) \ge 0.5
$$

则保留该边；否则删除。

---

## 1. 首先，LLM 不是直接说 knob1 连 burner1

假设系统已经重建出两个 knob：

$$
K_1,\ K_2
$$

以及两个 burner：

$$
B_1,\ B_2
$$

LLM 提供的是类别层面的功能知识：

$$
\text{knob}
\xrightarrow{\text{controls}}
\text{burner}
$$

并判断这种关系通常具有：

$$
1:1
$$

的 cardinality。

也就是说，一个 knob 通常对应一个 burner，一个 burner 通常也对应一个 knob。

然后 FunFact 枚举所有符合这个 relation template 的实例组合。

因此会产生四条 candidate edges：

$$
e_{11}: K_1 \rightarrow B_1
$$

$$
e_{12}: K_1 \rightarrow B_2
$$

$$
e_{21}: K_2 \rightarrow B_1
$$

$$
e_{22}: K_2 \rightarrow B_2
$$

注意，此时四条边都还不是最终确定的 functional edges，只是候选假设。

---

## 2. 每条 candidate edge 变成一个二值变量

FunFact 为每一条候选边定义一个二值随机变量：

$$
x_{11},x_{12},x_{21},x_{22} \in \{0,1\}
$$

例如：

$$
x_{11}=1
$$

表示：

$$
K_1
\xrightarrow{\text{controls}}
B_1
$$

这条关系成立。

而：

$$
x_{11}=0
$$

表示这条关系不存在。

这就是论文所谓的 **Dual Factor Graph**：

> 原 Scene Graph 中的一条 edge，在 Factor Graph 中变成一个 variable。

---

## 3. FunFact 不是独立决定每一条边

如果四条边分别独立判断，可能得到：

$$
P(x_{11}=1)=0.8
$$

$$
P(x_{21}=1)=0.75
$$

那么按照简单阈值，两条边都会被保留：

$$
K_1 \rightarrow B_1
$$

$$
K_2 \rightarrow B_1
$$

于是两个 knob 都控制同一个 burner。

但是我们已经知道：

$$
\text{knob} \rightarrow \text{burner}
$$

通常是一个 $1:1$ 的关系。

因此，这种连接方案并不合理。

FunFact 真正考虑的是联合概率：

$$
P(x_{11},x_{12},x_{21},x_{22})
$$

而不是分别计算：

$$
P(x_{11}),\quad
P(x_{12}),\quad
P(x_{21}),\quad
P(x_{22})
$$

这就是所谓的 **joint reasoning**。

---

# 4. 什么叫“整个连接方案”？

因为每个变量都只能取：

$$
0 \quad \text{或} \quad 1
$$

四条候选边一共有：

$$
2^4=16
$$

种可能的连接状态。

例如：

### 方案 A

$$
(x_{11},x_{12},x_{21},x_{22})
=
(1,0,0,1)
$$

表示：

```text
K1 ─── B1

K2 ─── B2
```

也就是：

$$
K_1 \rightarrow B_1
$$

$$
K_2 \rightarrow B_2
$$

---

### 方案 B

$$
(x_{11},x_{12},x_{21},x_{22})
=
(0,1,1,0)
$$

表示：

```text
K1 ─── B2

K2 ─── B1
```

---

### 方案 C

$$
(x_{11},x_{12},x_{21},x_{22})
=
(1,0,1,0)
$$

表示：

```text
K1 ─┐
    ├── B1
K2 ─┘

B2 没有 knob
```

---

### 方案 D

$$
(x_{11},x_{12},x_{21},x_{22})
=
(1,1,1,1)
$$

表示所有 knob 和所有 burner 都连接。

按照 $1:1$ 的功能常识，方案 A 和方案 B 通常比方案 C、D 更合理。

FunFact 的 Factor Graph 就是在给这些整体连接方案赋予不同的概率。

---

# 5. 第一个依据：Proximity

假设实际空间距离为：

$$
d(K_1,B_1)=10\text{ cm}
$$

$$
d(K_1,B_2)=50\text{ cm}
$$

$$
d(K_2,B_1)=45\text{ cm}
$$

$$
d(K_2,B_2)=12\text{ cm}
$$

显然，从空间距离看：

$$
K_1 \rightarrow B_1
$$

比：

$$
K_1 \rightarrow B_2
$$

更加合理。

FunFact 使用 proximity factor：

$$
\phi_{\text{prox}}
=
\exp\left(-\frac{d}{\lambda}\right)
$$

其中：

- $d$ 是 candidate edge 两端节点之间的距离；
- $\lambda$ 是距离尺度参数。

因此：

$$
d \downarrow
\quad\Rightarrow\quad
\phi_{\text{prox}} \uparrow
$$

即距离越近，这条候选 functional edge 得到的 prior 越高。

所以从 proximity 的角度：

$$
K_1-B_1
$$

以及：

$$
K_2-B_2
$$

更加有利。

但需要注意：

> FunFact 并不是直接选择距离最近的节点，而只是把距离作为概率推理中的一个 prior。

---

# 6. 第二个依据：Cardinality Constraint

除了距离以外，FunFact 还考虑功能关系的 cardinality。

这里：

$$
\text{knob}
\xrightarrow{\text{controls}}
\text{burner}
$$

被认为通常是：

$$
1:1
$$

的关系。

对于 $K_1$，它可能连接：

$$
B_1,\ B_2
$$

因此 $K_1$ 当前的连接数量为：

$$
d_{K_1}=x_{11}+x_{12}
$$

理想情况下：

$$
d_{K_1}=1
$$

论文定义 cardinality factor：

$$
\phi_{\text{card}}(\mathcal{X}_n)
=
\begin{cases}
b^{d_n-1}, & d_n \ge 1,\\
b^2, & d_n=0,
\end{cases}
\qquad 0<b<1
$$

其中：

$$
d_n=\sum_{x\in\mathcal{X}_n}x
$$

表示某个节点当前连接了多少条 functional edges。

---

## 当 $d_n=1$

有：

$$
\phi_{\text{card}}=1
$$

不进行惩罚。

这正是系统最偏好的情况。

---

## 当 $d_n=2$

有：

$$
\phi_{\text{card}}=b
$$

由于：

$$
0<b<1
$$

因此该配置会受到惩罚。

---

## 当 $d_n=0$

有：

$$
\phi_{\text{card}}=b^2
$$

同样会受到惩罚。

因此这个 factor 最偏好的是：

$$
\boxed{d_n=1}
$$

也就是：

> 对于一个 $1:1$ functional relation，一个节点最好恰好连接一个对应节点。

---

# 7. 为什么方案 A 会得到较高概率？

考虑：

$$
A=(1,0,0,1)
$$

即：

```text
K1 ─── B1

K2 ─── B2
```

首先，从 proximity 看：

$$
K_1-B_1
$$

距离较近，

$$
K_2-B_2
$$

距离也较近。

其次，从 cardinality 看：

$$
d_{K_1}=1
$$

$$
d_{K_2}=1
$$

$$
d_{B_1}=1
$$

$$
d_{B_2}=1
$$

因此所有节点都符合 $1:1$ 的关系约束。

所以该方案同时获得：

- 较好的 proximity score；
- 较好的 cardinality score。

最终整体概率较高。

---

## 方案 C 为什么会被压低？

考虑：

$$
C=(1,0,1,0)
$$

即：

```text
K1 ─┐
    ├── B1
K2 ─┘

B2 没有连接
```

此时：

$$
d_{B_1}=2
$$

而：

$$
d_{B_2}=0
$$

因此：

$$
\phi_{\text{card}}(B_1)<1
$$

同时：

$$
\phi_{\text{card}}(B_2)<1
$$

这个整体配置就会被降低概率。

这正是所谓：

$$
\boxed{
\text{一条边的判断会影响其他边}
}
$$

例如，如果：

$$
K_1 \rightarrow B_1
$$

已经具有很高的概率，那么由于 $1:1$ constraint：

$$
K_2 \rightarrow B_1
$$

的概率就应该相应降低。

---

# 8. Joint Distribution 到底表示什么？

FunFact 建立的联合概率可以写成：

$$
P(X)
=
\frac{1}{Z}
\prod_x
\phi_{\text{prox}}(x)
\prod_f
\phi_{\text{card}}(\partial f)
$$

其中：

$$
X=
(x_{11},x_{12},x_{21},x_{22},\ldots)
$$

表示整个 functional graph 的一种连接配置。

因此可以粗略理解为：

$$
\boxed{
\text{整体方案得分}
=
\text{几何合理性}
\times
\text{结构合理性}
}
$$

其中：

$$
Z
$$

是归一化常数，使得：

$$
\sum_XP(X)=1
$$

所以：

$$
P(X)
$$

描述的不是某一条边，而是：

> 整个候选 functional graph 处于某种连接状态的概率。

---

# 9. 最后为什么还要计算 Marginal Probability？

虽然 Factor Graph 建模的是：

$$
P(X)
$$

但最终我们通常想知道：

> $K_1 \rightarrow B_1$ 这一条边到底有多大概率存在？

因此需要计算：

$$
P(x_{11}=1)
$$

理论上：

$$
P(x_{11}=1)
=
\sum_{X:x_{11}=1}P(X)
$$

也就是说：

> 把所有满足 $x_{11}=1$ 的整体连接方案的概率全部加起来。

这个操作叫做：

$$
\boxed{\text{Marginalization}}
$$

得到的：

$$
P(x_{11}=1)
$$

就是这条 edge 的 **marginal probability**。

例如：

$$
P(x_{11}=1)=0.92
$$

它的含义不是：

> 因为 $K_1$ 和 $B_1$ 离得最近，所以距离模型给了 0.92。

而是：

> 在综合考虑其他所有 candidate edges、空间距离和 cardinality constraint 后，这条边存在的 posterior belief 为 0.92。

---

# 10. Belief Propagation 在这里干什么？

理论上，可以直接枚举所有可能的 $X$。

对于四条 candidate edges：

$$
2^4=16
$$

还比较简单。

但如果有 20 条候选边：

$$
2^{20}=1,048,576
$$

如果有 100 条候选边：

$$
2^{100}
$$

就几乎不可能直接枚举。

因此 FunFact 在 Factor Graph 上使用 **Belief Propagation，BP**。

Factor Graph 中大致存在：

```text
Proximity Factor
       │
      x11
       │
Cardinality Factor
       │
      x12
       │
Proximity Factor
```

不同 variable 和 factor 之间不断传递 message。

最终可以得到每条候选边的 marginal distribution：

$$
P(x_i=0)
$$

和：

$$
P(x_i=1)
$$

所以 BP 的作用可以理解成：

$$
\boxed{
P(X)
\rightarrow
P(x_i)
}
$$

即：

> 从整个联合概率模型中，高效地求出每条候选边自己的 posterior probability。

---

# 11. 最终到底怎么决定“连还是不连”？

假设 Belief Propagation 得到：

$$
P(x_{11}=1)=0.92
$$

$$
P(x_{12}=1)=0.04
$$

$$
P(x_{21}=1)=0.06
$$

$$
P(x_{22}=1)=0.89
$$

论文采用阈值：

$$
\tau=0.5
$$

于是：

$$
P(x_{11}=1)=0.92>0.5
$$

所以保留：

$$
K_1
\xrightarrow{\text{controls}}
B_1
$$

而：

$$
P(x_{12}=1)=0.04<0.5
$$

所以删除：

$$
K_1
\xrightarrow{\text{controls}}
B_2
$$

同理：

$$
P(x_{21}=1)=0.06<0.5
$$

删除：

$$
K_2
\xrightarrow{\text{controls}}
B_1
$$

而：

$$
P(x_{22}=1)=0.89>0.5
$$

保留：

$$
K_2
\xrightarrow{\text{controls}}
B_2
$$

最终得到：

```text
K1 ─── controls ───→ B1

K2 ─── controls ───→ B2
```

---

# 12. 完整流程

FunFact 中 functional edge 的决策过程可以总结为：

```text
LLM 提出 relation template
        │
        ▼
knob --controls--> burner
        │
        ▼
枚举所有实例组合
        │
        ▼
Ki --controls--> Bj
        │
        ▼
每条 candidate edge 变成二值变量
        │
        ▼
xij ∈ {0,1}
        │
        ├───────────┐
        ▼           ▼
Proximity      Cardinality
  Prior         Constraint
        │           │
        └─────┬─────┘
              ▼
           P(X)
              │
              ▼
     Belief Propagation
              │
              ▼
        P(xij = 1)
              │
              ▼
      threshold = 0.5
        ┌─────┴─────┐
        ▼           ▼
    ≥ 0.5         < 0.5
    保留边         删除边
```

数学上就是：

$$
\text{Candidate Edges}
\rightarrow
P(X)
\rightarrow
P(x_i=1)
\rightarrow
\text{Threshold}
\rightarrow
\text{Final Functional Graph}
$$

---

# 13. 一个容易混淆的地方：它不是直接求一个最优匹配

FunFact 并不是简单计算：

$$
X^*
=
\arg\max_XP(X)
$$

然后把 $X^*$ 中所有值为 1 的边作为最终结果。

它采用的是：

$$
P(X)
\rightarrow
P(x_i=1)
\rightarrow
\text{threshold}
$$

也就是先求每条边的 marginal probability。

因此它不仅能够输出最终 functional graph，还能保留 uncertainty。

例如：

$$
P(K_1\rightarrow B_1)=0.92
$$

而：

$$
P(K_1\rightarrow B_2)=0.46
$$

按照：

$$
\tau=0.5
$$

第二条不会进入最终离散 Scene Graph。

但 $0.46$ 仍然告诉系统：

> 目前这条关系并不是完全不可能，只是当前证据还不足以支持它。

这就是 **Probabilistic Functional 3D Scene Graph** 中 “Probabilistic” 的主要意义。