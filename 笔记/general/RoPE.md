一种位置编码方式

本文对于位置编码我认为有朴素的假设：位置编码用于transformer来计算上下文注意力，越远的文本在此刻的注意力应该越小。并且，位置编码不能破坏词向量语义本身。
## 复数
- **欧拉公式**：联系起来复数和可计算的桥梁
$$e^{i\theta} = \cos\theta + i\sin\theta \tag{1}$$
- **向量旋转**：矩阵表达形式（这是根据欧拉公式推导的），物理意义时将向量`<a,b>`逆时针旋转  $\theta$  度数，得到模长不变但是角度变化的新向量。下面是等价的两个式子，为了方便表达，我们会通常使用第一个式子。
$$(a,b)e^{i\theta} = (a^{'},b^{'})\tag{2}$$
$$\begin{bmatrix} a' \\ b' \end{bmatrix} = \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} a \\ b \end{bmatrix}\tag{2}$$
- **复数向量的内积**：我们往往只需要计算向量的内积，可以数学推导得到向量内积等于向量一和共轭向量二乘积的**实部**，Re是取实部的意思。<>表示内积
$$<z_1,z_2> = Re[z_1,\overline{z_2}]\tag{3}$$

## 2维推导
让我们假设词向量是一个二维向量。
- 核心思想：我们想要两个向量Xm和Xn在位置编码  f 作用之后，能够显现与m-n相关的因素，m-n是能够直接展现相关位置的直观表达式。
$$<f_q(x_m,m),f_k(x_n,n)> = g(x_m,x_n,m-n)\tag{main1}$$
- 数学逻辑：不难发现exp(x)是相乘变成相加的绝佳选择，推导包含了**共轭，转置**的知识点
$$f_q(x_m,m) = (W_qx_m)e^{im\theta}\tag{main2} = q_me^{im\theta}$$
$$g(x_m,x_n,m-n) = Re[q_m\overline{k_n}e^{i(m-n)\theta}]\tag{main3}$$
- 数学逻辑的向量表示形式：
$$f_q(x_m,m)= \begin{bmatrix} \cos m\theta & -\sin m\theta \\ \sin m\theta & \cos m\theta \end{bmatrix} \begin{bmatrix} q_{m,1} \\ q_{m,2} \end{bmatrix}\tag{4}$$
$$g(x_m,x_n,m-n)= [q_{m,1}\ \ q_{m,2}]\begin{bmatrix} \cos m\theta & \sin m\theta \\ -\sin m\theta & \cos m\theta \end{bmatrix}\begin{bmatrix} \cos n\theta & -\sin n\theta \\ \sin n\theta & \cos n\theta \end{bmatrix} \begin{bmatrix} k_{n,1} \\ k_{n,2} \end{bmatrix}  $$
$$=[q_{m,1}\ \ q_{m,2}]\begin{bmatrix} \cos (m-n)\theta & -\sin (m-n)\theta \\ \sin (m-n)\theta & \cos (m-n)\theta \end{bmatrix} \begin{bmatrix} k_{n,1} \\ k_{n,2} \end{bmatrix} \tag{5}$$
- 因此，我们总结为：对于二维向量，通过上述位置编码，两个向量在点积之后，能够体现出位置关系m-n。解释为物理意义，就是在向量原来基础上，拉开了(m-n)θ的夹角距离，从而体现出相对位置。

## 多维拓展
真实的词向量通常是512维或768维，我们不能直接应用2D旋转：but
从数学上，d维空间的旋转群可以分解为多个二维旋转群的直积。换句话说，**任意高维旋转都能拆分为若干个独立的二维平面上的旋转，且这些二维旋转互不干扰** ，这是我们的理论基础。

- **解决办法**：将高维向量分解成多个2D平面，将512维向量转化为256个二维平面，然后每个平面独立使用不同的旋转频率进行旋转。
$$f_{\{q,k\}}(x_m,m) = R_{\Theta,m} W_{\{q,k\}}x_m \tag{6}$$
$$\mathbf{R}_{\Theta, m} = \mathrm{diag}\left(\begin{pmatrix} \cos(m\theta_0) & -\sin(m\theta_0) \\ \sin(m\theta_0) & \cos(m\theta_0)\end{pmatrix},\begin{pmatrix} \cos(m\theta_1) & -\sin(m\theta_1) \\ \sin(m\theta_1) & \cos(m\theta_1)\end{pmatrix},\ldots,\begin{pmatrix} \cos(m\theta_{d/2}) & -\sin(m\theta_{d/2}) \\ \sin(m\theta_{d/2}) & \cos(m\theta_{d/2}) \end{pmatrix}\right)$$
$$({R}_{\Theta, n})^T{R}_{\Theta, m} = {R}_{\Theta, m-n} \tag{7}$$
- $\theta$  含义：保证相对衰减性
$$\theta_i = 10000^{-2i/d}\tag{8}$$
![相对衰减性的数学推导|500](file-20251105171609617.png)
![衰减性图表|400](file-20251105171633229.png)

从图中我们可以看到随着相对距离的变大，内积结果有衰减趋势的出现。因此，选择$\theta_i = 10000^{-2i/d}$，确实能带来一定的远程衰减性（这种衰减性一定程度上减少了注意力的权重，解释为文本天然性的关注靠近的上下文）。论文中还试过以 $\theta_i = 10000^{-2i/d}$为初始化，将  $\theta_i$  视为可训练参数，然后训练一段时间后发现并没有显著更新，因此干脆就直接固定了。
## 高效计算
矩阵具有稀疏性，不好计算，改进如下：那个奇怪的符号表示**逐位相乘**

$$\boldsymbol{R}_{\theta, m}^d \boldsymbol{x} = \begin{pmatrix} x_0 \\ x_1 \\ x_2 \\ x_3 \\ \vdots \\ x_{d-2} \\ x_{d-1} \end{pmatrix} \otimes \begin{pmatrix} \cos m\theta_0 \\ \cos m\theta_0 \\ \cos m\theta_1 \\ \cos m\theta_1 \\ \vdots \\ \cos m\theta_{d/2-1} \\ \cos m\theta_{d/2-1} \end{pmatrix} + \begin{pmatrix} -x_1 \\ x_0 \\ -x_3 \\ x_2 \\ \vdots \\ -x_{d-1} \\ x_{d-2} \end{pmatrix} \otimes \begin{pmatrix} \sin m\theta_0 \\ \sin m\theta_0 \\ \sin m\theta_1 \\ \sin m\theta_1 \\ \vdots \\ \sin m\theta_{d/2-1} \\ \sin m\theta_{d/2-1} \end{pmatrix} \tag{9}$$

总结来说，RoPE 的 self-attention 操作的流程是：对于 token 序列中的每个词嵌入向量，首先计算其对应的 query 和 key 向量，然后对每个 token 位置都计算对应的旋转位置编码，接着对每个 token 位置的 query 和 key 向量的元素按照**两两一组**应用旋转变换，再计算 query 和 key 之间的内积得到 self-attention 的计算结果。

最后——只有通过实打实的代码实现才能真正理解RoPE的运行。

## 参考资料
1. [知乎文章](https://zhuanlan.zhihu.com/p/647109286)

苏剑林大佬的博客:
