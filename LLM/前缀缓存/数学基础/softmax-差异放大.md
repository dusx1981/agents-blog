按照 **Softmax 格式**（数学分布展开与概率映射），`exp` 的“选择性放大”作用严格表达如下：

$$
\begin{aligned}
\text{Softmax}(\mathbf{z})_i &= \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}} \\
&\Downarrow \text{（核心差异放大机制）} \\
\text{对于任意两项 } i, k \quad \frac{p_i}{p_k} &= \frac{e^{z_i}}{e^{z_k}} = e^{\,z_i - z_k}
\end{aligned}
$$

---

**在该格式下的量化对比（线性 vs 指数）：**

设原始逻辑值（Logits）为 $\mathbf{z} = [2, 4, 6]$：

1. **若为线性放大（无 exp）**，权重占比为：
   $$
   \frac{2}{12} : \frac{4}{12} : \frac{6}{12} = 16.7\% : 33.3\% : 50.0\%
   $$
   （最大值仅比最小值多 $3$ 倍权重，差距温和）

2. **经 exp 指数级放大后**，输出概率分布为：
   $$
   \frac{e^2}{e^2+e^4+e^6} : \frac{e^4}{e^2+e^4+e^6} : \frac{e^6}{e^2+e^4+e^6}
   $$
   化简比例：
   $$
   e^2 : e^4 : e^6 = 1 : e^2 : e^4 \approx 1 : 7.39 : 54.60
   $$
   最终概率：
   $$
   \mathbf{p} \approx [0.0159, 0.1173, 0.8668]
   $$

---

**结论（Softmax 格式下的显式作用）：**

$$
\boxed{
\begin{aligned}
&\text{1. 放大差异：} && \text{微小逻辑差 } \Delta \text{ 被映射为 } e^{\Delta} \text{ 倍的概率比（非 } \Delta \text{ 倍）。} \\
&\text{2. 选择性抑制：} && \text{最小项（如 } e^2 \text{）概率被压缩至 } < 2\% \text{，接近“遗忘”。} \\
&\text{3. 选择性聚焦：} && \text{最大项（如 } e^6 \text{）概率被锐化至 } > 86\% \text{，成为绝对主导。}
\end{aligned}
}
$$

即：**`exp` 将线性空间中的“领先优势”转化为指数空间中的“统治地位”，实现注意力机制中对高激活特征的精准锁定。**