## Exponential Time Differencing

在处理具有*空间周期性*的偏微分方程时，我们常常使用傅立叶变换将其转换成谱空间内的 ODE 系统，在谱空间内，求导算子变为标量乘法，极大的便利了数值求解。

但谱空间的分解也带来了另一个问题，由于谱空间上的振幅多样，导致构成的 ODE 系统具有刚性（stiffness），因为第 $n$ 个波对应的时间尺度为 $O(n^{-m})$，$m$ 为其对应的空间导数。

#### Example

考虑方程
\$$
i\frac{\partial u}{\partial t}=-\frac{1}{2}\frac{\partial^2 u}{\partial x^2}-|u|^2u
$$
进行傅立叶变换，$\hat{u}(n)=\int_{-\infty}^{\infty}u(x)e^{-2\pi ikx}\text{d}x$，对上面的方程而言，我们如果只考虑线性部分，线性部分变成了一个单纯的标量乘法。
$$
\frac{\partial \hat{u}}{\partial t}=-i\frac{k^2}{2}
$$
其每一步都需要变化相当于 $O(k^2)$ 的值，由于不同的 mode 下，$k$ 的值覆盖了很大的一块区间，导致谱方法出现了刚性。

## 解决手段——将线性部分精确处理

对于经过谱方法变换过的 ODE 系统，通常都有下面的形式。
$$
u'=cu+F(u,t)
$$
其中 $F$ 为非线性部分，$c$ 为谱空间内的求导算子 $\mathcal{L}$。

对两侧乘以积分因子 $e^{-ct}$，我们有
$$
\begin{align}
u'e^{-ct}-ce^{-ct}&=e^{-ct}F(u,t)\\
(ue^{-ct})'&=e^{-ct}F(u,t)\\
u(t_{n+1})&=e^{ch}u(t_n)+e^{ch}\int_0^h e^{-c\tau}F(u(t_n+\tau),t_n+\tau)\text{d}\tau
\end{align}
$$
注意到上式是精确的，因此重点只在于右侧的非线性部分的积分近似。

