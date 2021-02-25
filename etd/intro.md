## Exponential Time Differencing

在处理具有*空间周期性*的偏微分方程时，我们常常使用傅立叶变换将其转换成谱空间内的 ODE 系统，在谱空间内，求导算子变为标量乘法，极大的便利了数值求解。

但谱空间的分解也带来了另一个问题，由于谱空间上的振幅多样，导致构成的 ODE 系统具有刚性（stiffness），因为第 $n$ 个波对应的时间尺度为 $O(n^{-m})$，$m$ 为其对应的空间导数。

#### Example

考虑方程
\$\$
i\frac{\partial u}{\partial t}=-\frac{1}{2}\frac{\partial^2 u}{\partial x^2}-|u|^2u
\$\$
进行傅立叶变换，$\hat{u}(n)=\int_{-\infty}^{\infty}u(x)e^{-2\pi ikx}\text{d}x$，对上面的方程而言，我们如果只考虑线性部分，线性部分变成了一个单纯的标量乘法。
\$\$
\frac{\partial \hat{u}}{\partial t}=-i\frac{k^2}{2}
\$\$
其每一步都需要变化相当于 $O(k^2)$ 的值，由于不同的 mode 下，$k$ 的值覆盖了很大的一块区间，导致谱方法出现了刚性。

## 解决手段——将线性部分精确处理

对于经过谱方法变换过的 ODE 系统，通常都有下面的形式。
\$\$
u'=cu+F(u,t)
\$\$
其中 $F$ 为非线性部分，$c$ 为谱空间内的求导算子 $\mathcal{L}$。

对两侧乘以积分因子 $e^{-ct}$，我们有
\$\$
\begin{align}
u'e^{-ct}-ce^{-ct}&=e^{-ct}F(u,t)\\
(ue^{-ct})'&=e^{-ct}F(u,t)\\
u(t_{n+1})&=e^{ch}u(t_n)+e^{ch}\int_0^h e^{-c\tau}F(u(t_n+\tau),t_n+\tau)\text{d}\tau
\end{align}
\$\$
注意到上式是精确的，因此重点只在于右侧的非线性部分的积分近似。
## 最基础的格式——ETD1
假设我们采用最基础的近似方式，将 $F$ 在节点间的值看成常数，$F=F_n+O(h),t\in[t_n,t_{n+1}]$。
\$\$
u_{n+1}=u(t_n)e^{ch}+F_n\frac{e^{ch}-1}{c}
\$\$
其他格式例如 ETD2，ETDRK4 等，都是对非线性部分采用不同的数值积分方法进行处理。

## 极限形式处理——Limiting Form
注意到，在进行 ETD 数值格式计算时，会出现谱空间内求导算子 $\mathcal{L}=0$ 的情况。这直接导致在右侧的非线性部分中出现分子为 $0$。对于这个问题，我们利用洛必达法则，对分式上下同时求导，得到极限形式。
\$\$
\begin{align}
\lim_{c\to0}\frac{e^{ch}-1}{c}=h
\end{align}
\$\$
当然对于不同的 ETD 格式我们会推导出不同的极限形式，后面我们会讨论相关的处理。

## MATLAB 代码
```matlab
N = 1024;
dt = 0.001;
t_max = 10;
step_max = round(t_max/dt);

output_perd = 10;
L = 50;
h = L/N;
n = (-N/2:1:N/2-1)';
x = n * h;
u = exp(1i*x).*sech(x);
k = 2*n*pi/L;

M = moviein(step_max/output_perd);
c = -1i*k.*k/2;

for s = 1:step_max
    
    u_mode = fftshift(fft(u));
    uN_mode = fftshift(fft(abs(u.*u).*u));
    expch = exp(dt*c);
    
    ulim = u_mode(513) + dt * 1i .* uN_mode(513);
    u_mode = expch .* u_mode + 1i*uN_mode.*(expch-1)./c;
    u_mode(513) = ulim;
    u = ifft(ifftshift(u_mode));
    
    energy = sum(abs(u).^2);
    mass = sum(abs(u));
    fprintf("Energy: %.2f\nMass: %.2f\n", energy, mass);
        
    if rem(s, output_perd) == 0
        frame_idx = s / output_perd;
        plot3(x, real(u), imag(u));
        view(0, 90);
        M(frame_idx) = getframe;
    end
end
video=VideoWriter('ETD1.avi');
open(video);
video.writeVideo(M);
close(video);
fprintf("Done")
```