# 25年电赛G题
存在一个小问题：硬件电路将输入信号人为放大了一倍，属于当时硬件设计失误，如要复现请修改代码中的逻辑
代码性能指标存在以下优化点：
1.DAC 零阶保持效应 (ZOH) 的频域逆补偿 (Hardware Compensation)
这是保障高频合成波形“无失真”的核心。在离散信号转换为连续物理信号时，片内 DAC 并不是输出理想的冲激脉冲，而是在采样间隔内保持电压不变（零阶保持器特性）。这在频域上等效于引入了一个额外的低通滤波器：$$H_{ZOH}(f) = e^{-j \pi \frac{f}{f_s}} \frac{\sin(\pi f / f_s)}{\pi f / f_s}$$当高次谐波逼近奈奎斯特频率时，ZOH 效应会导致极其严重的幅度衰减和相位滞后。为此，在频域调制阶段，系统在乘以模型增益 $|H_{model}(\omega)|$ 的基础上，主动引入了针对物理硬件的 ZOH 逆补偿因子，提前拉高高频幅值并超前补偿相位：$$|Y_h| = |X_h| \cdot |H_{model}(h \cdot f_0)| \cdot \left[ \frac{\pi h f_0 / f_s}{\sin(\pi h f_0 / f_s)} \right]$$$$\angle Y_h = \angle X_h + \angle H_{model}(h \cdot f_0) + \pi \frac{h \cdot f_0}{f_s}$$这一处理跨越了纯数学算法与实际物理硬件的鸿沟。
算法运算加速存在以下优化点：
1.基于 Goertzel 算法的谐波极速提取 (Forward Decomposition)利用 TIM4 捕获外部信号的基频 $f_0$ 后，系统仅需关注基波及前 9 次谐波。相比于执行全量 FFT 带来的算力浪费，系统对这 10 个特定谐波独立并行执行 Goertzel 算法。每次采样点迭代仅需一次乘法和两次加法，在微秒级时间内即可精准提取出输入信号 $X(\omega)$ 的谐波幅值 $|X_h|$ 和相位 $\angle X_h$。
2.基于耦合型递推振荡器的时域合成 (Inverse Harmonic Synthesis)
在获得补偿后的目标谐波分量 $|Y_h|$ 与 $\angle Y_h$ 后，需进行离散傅里叶级数合成（IDFS）。传统的合成方程需在每次循环中频繁调用耗时的 math.h 三角函数：$$y[n] = DC + \sum_{h=1}^{10} |Y_h| \cdot \cos\left( 2\pi h f_0 \frac{n}{f_s} + \angle Y_h \right)$$为确保 DMA 缓冲区能在极短的中断周期内被填满，本方案将其转化为耦合型递推振荡器 (Coupled-form Oscillator)。仅在初始时刻计算状态转移矩阵，后续所有波形点的生成完全依靠纯硬件乘加运算（MAC）：$$\begin{bmatrix}
y_1[n] \\
y_2[n] 
\end{bmatrix}
=
\begin{bmatrix}
\cos(\omega_h) & -\sin(\omega_h) \\
\sin(\omega_h) & \cos(\omega_h)
\end{bmatrix}
\begin{bmatrix}
y_1[n-1] \\
y_2[n-1] 
\end{bmatrix}$$结合 ARM 处理器的 FPU 指令加速，彻底打通了高速实时重构的算力瓶颈。
