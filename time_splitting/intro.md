## Time Splitting Method
对于经过谱方法进行空间离散的方程，通常有下列形式
\$\$
    \frac{\partial u}{\partial t}=\mathcal{L}u+\mathcal{N}u
\$\$
一个最直观的方法便是对线性部分和非线性部分分别处理，通过引入 Splitting Error（算子交换导致的误差）。便利的处理该方程。

## 非线性部分处理
首先考虑非线性部分 $\mathcal{N}$，假设忽略线性部分，即 $u_{xx}=0$，我们有
\$\$
    \frac{\partial u}{\partial t}=\mathcal{N}u
\$\$
我们可以直接写出这个 ODE 的解。
\$\$
    u(x,\tau+\Delta t)=\exp(\Delta t\mathcal{N})u(x,\tau)
\$\$

## 线性部分处理
线性部分如下所示
\$\$
    \frac{\partial u}{\partial t}=\mathcal{L}u
\$\$
利用傅立叶变换我们有下面的结论
\$\$
    \frac{\partial \hat{u}}{\partial t}=\mathcal{F}(\frac{\partial^2 u}{\partial x^2})=-k^2x
\$\$
在谱空间内解该方程，再变换回物理空间，我们得到下面的式子。
\$\$
u(x,\tau+\Delta t)=\mathcal{F}^{-1}(\exp(-k^2\Delta t)\mathcal{F}(\exp(\Delta t\mathcal{N})u(x,\tau)))
\$\$

## 分解误差
该方法会引入一个额外的分解误差（Splitting Error）。考虑原问题
\$\$
    \frac{\partial u}{\partial t}=\mathcal{L}u+\mathcal{N}u
\$\$
该方程的解析解为
\$\$
    u(t)=e^{(\mathcal{L}+\mathcal{N})t}u_0
\$\$
但我们如果采用 Time Splitting 处理，实际得到的解为
\$\$
    u(t)=e^{\mathcal{L}t}e^{\mathcal{N}t}u_0
\$\$
进行泰勒展开，我们有
\$\$
    e^{(\mathcal{L}+\mathcal{N})t}=1+(\mathcal{L}+\mathcal{N})t+\frac{(\mathcal{L}+\mathcal{N})^2}{2}t^2+\cdots
\$\$
而对于分解后的解，我们有
\$\$
    e^{\mathcal{L}t}e^{\mathcal{N}t}=(1+\mathcal{L}t+\frac{\mathcal{L}^2}{2}t^2+\cdots)(1+\mathcal{N}t+\frac{\mathcal{N}^2}{2}t^2+\cdots)
\$\$
可以注意到的是，以二次项为例，前者拥有的二次项为
\$\$
    \frac{\mathcal{L}^2+\mathcal{N}^2+\mathcal{LN}+\mathcal{NL}}{2}
\$\$
后者的二次项为
\$\$
    \frac{\mathcal{L}^2+\mathcal{N}^2+2\mathcal{LN}}{2}
\$\$
当 $\mathcal{L}$ 和 $\mathcal{N}$ 可以交换时（commute），误差为 $0$，但一般这种情况不会出现，因此一般都会引入一个二阶分解误差。
## MATLAB 程序（含傅立叶变换）
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

for s = 1:step_max
    energy = sum(abs(u).^2);
    mass = sum(abs(u));
    fprintf("Energy: %.2f\nMass: %.2f\n", energy, mass);
    u = exp(dt/2*1i*(abs(u.*u))).*u;
    v = fftshift(fft(u));
    v = exp(-dt*1i*k.*k/2).*v;
    u = ifft(fftshift(v));
    u = exp(dt/2*1i*(abs(u.*u))).*u;
    if rem(s, output_perd) == 0
        frame_idx = s / output_perd;
        plot(x, real(u));
        M(frame_idx) = getframe;
    end
end
video=VideoWriter('S3FM.avi');
open(video);
video.writeVideo(M);
```