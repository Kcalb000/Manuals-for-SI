# Manuals-for-SI
Technical documents for super intelligence era
# 多相 VR 同步 Buck 环路控制详解

> **范围**：多相交错同步 Buck VR/VRM 的电压型控制、峰值电流模式、平均电流模式和恒定导通时间（CoT/COT）。覆盖原理、平均模型、传递函数、设计计算、控制框图、优缺点与环路节点波形。
>
> **模型边界**：除特别说明外，公式建立在 CCM、小信号平均模型和理想开关假设上。真实控制器还包括均流、限流、死区、软启动、延迟、采样量化、相位管理等。CoT 是事件触发控制，线性传递函数只能在特定稳态工作点附近近似使用。

## 1. 系统拓扑与工作原理

### 1.1 单相同步 Buck 功率回路

![多相同步 Buck 功率拓扑。每一相均包含高边/低边 MOSFET 与电感，多个电感输出并联。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/01_multiphase_buck_topology.svg)

**图 1.** 四相交错同步 Buck VR 功率拓扑。图中各相是单相功率回路的复制，绿色虚线为输出反馈；输入去耦、采样网络、驱动死区和寄生参数未画出。

高边导通时，电感两端电压约为 $V_{in}-V_o$，电流上升；低边同步整流导通时，电感两端电压约为 $-V_o$，电流下降。理想 CCM 稳态满足伏秒平衡：

$$D(V_{in}-V_o)+(1-D)(-V_o)=0$$

因此：

$$D\approx\frac{V_o}{V_{in}}$$

### 1.2 多相交错结构

N 相功率级并联到共同输出，控制器将开关时钟错开 $360^\circ/N$。各相电流在输出端相加：

多相并联路径和各相开关节点见图 1。每相时钟彼此错相，电感电流在公共输出端相加。

每相平均电流与开关相位关系：

$$I_{L,avg}\approx\frac{I_o}{N}, \qquad \Delta t_{phase}=\frac{1}{Nf_s}$$

每相频率是 $f_s$；合成纹波常含有 $Nf_s$ 的主要分量，但谐波和纹波幅值仍取决于占空比、相数及控制器实现。多相设计同时要求稳压和均流：

$$I_{L,1}\approx I_{L,2}\approx\cdots\approx I_{L,N}$$

## 2. 功率级小信号模型

### 2.1 单相 Buck 控制到输出传递函数

忽略电感 DCR 和电容 ESR 时，连续导通模式下的占空比到输出电压传递函数为：

$$G_{vd}(s)=\frac{\hat v_o(s)}{\hat d(s)}\approx\frac{V_{in}}{LCs^2+\frac{L}{R}s+1}$$

LC 双极点和品质因数：

$$f_{LC}=\frac{1}{2\pi\sqrt{LC}}, \qquad Q=R\sqrt{\frac{C}{L}}$$

电容 ESR 为 $r_C$ 时，常见近似模型为：

$$G_{vd}(s)\approx V_{in}\frac{1+s r_C C}{1+s(\frac{L}{R}+r_C C)+s^2LC}$$

ESR 零点：

$$f_{ESR}=\frac{1}{2\pi r_C C}$$

若内电流环足够快，电流命令到输出电压常近似为输出阻抗：

$$Z_o(s)\approx\frac{1+s r_C C}{sC+1/R}$$

图 2 的控制结构总览展示了占空比调制、功率级和输出反馈的关系；$G_{vd}(s)$ 是占空比至输出的功率级通道，$Z_o(s)$ 是负载电流扰动至输出电压的通道。

以上是平均模型。精确设计应纳入电感 DCR、MOSFET 导通电阻、电容 ESL、负载动态、PWM/ADC 延迟和控制器的采样行为。

### 2.2 每相与合成电流纹波

单相电感电流纹波：

$$\Delta I_{L,pp}=\frac{(V_{in}-V_o)D}{Lf_s}=\frac{V_o(1-D)}{Lf_s}$$

N 相相同参数，且 $D<1/N$ 时，理想交错后的总电流纹波近似：

$$\Delta I_{\Sigma,pp}\approx\frac{V_{in}}{Lf_s}D(1-ND)$$

本式仅适用于相同 $L$、相同占空比、理想 CCM 和 $D<1/N$。跨越 $k/N$ 的占空比区间时应分段推导；参数失配时使用开关级仿真或实测。

![四相电感电流及其合成电流纹波示意。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/04_interleaving_and_load_step.svg)

**图 2.** 上半部分展示四相电流错相叠加后的纹波抵消；下半部分展示负载阶跃时电容电流与电感总电流的接力过程。曲线为定性示意，不能用于直接读取设计数值。

## 3. 四种控制方式对照

| 控制方式 | 直接调节对象 | 典型调制 | 优点 | 主要风险 |
|---|---|---|---|---|
| 电压型 | 输出电压 | 固定频率 PWM 占空比 | 结构直观，固定频率，采样简单 | LC 双极点补偿难，均流需另做 |
| 峰值电流模式 | 电感峰值电流 | 峰值比较器 + PWM 锁存 | 逐周期限流快，电压外环简单 | 高占空比次谐波，噪声敏感，峰均误差 |
| 平均电流模式 | 平均电感电流 | 电流误差放大器 + PWM | 均流准确、电流命令线性 | 环路多、采样精度要求高 |
| CoT | 触发时刻/导通脉冲 | 固定或自适应 Ton | 负载瞬态快，专用控制器结构精简 | 频率变化，低 ESR 纹波注入，事件系统难线性化 |

![电压型、峰值电流、平均电流和 CoT 控制的并列框图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/02_control_architectures.svg)

**图 3.** 四种控制方式的控制路径对照。A 为电压型，B 为峰值电流模式，C 为平均电流模式，D 为 CoT。箭头表示主要信号流，绿色虚线表示反馈或采样路径。

## 4. 电压型控制（Voltage-Mode Control）

### 4.1 控制框图与传递函数

![电压型控制的独立放大框图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/02a_voltage_mode_control.svg)

**图 3A.** 外电压环经补偿器驱动固定频率 PWM；输出分压反馈回到误差比较端。

环路增益：

$$T_v(s)=G_c(s)H(s)F_{PWM}(s)G_{vd}(s)$$

锯齿波峰峰值为 $V_{ramp}$ 时，理想 PWM 增益约为：

$$F_{PWM}\approx\frac{1}{V_{ramp}}\quad[duty/V]$$

Buck 的 LC 双极点常要求 Type III 补偿：

$$G_c(s)=K\frac{(1+s/\omega_{z1})(1+s/\omega_{z2})}{s(1+s/\omega_{p1})(1+s/\omega_{p2})}$$

两个零点用于提升 LC 双极点附近相位；交越频率初始常取 $f_c<f_s/10$，再按延迟、噪声和目标瞬态调整。高频极点用于衰减开关噪声。需通过模型、环路注入和瞬态测试核验裕度。

### 4.2 节点波形示例

![四种控制方式的关键控制节点和功率级波形。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/03_control_node_waveforms.svg)

**图 4.** 图中 A 区是电压型控制：锯齿波固定，VCOMP 改变 PWM 占空比；负载变化先反映在 VOUT/VFB，随后控制量和相电流响应。

| 节点 | 观察重点 |
|---|---|
| $V_{FB}$ | 输出缩放值、噪声和纹波 |
| $V_{COMP}$ | 补偿器输出及饱和情况 |
| $V_{RAMP}$ | 固定频率、幅度和线性度 |
| PWM / Gate | 占空比与相位关系 |
| $V_{SW}$ | 死区、过冲、寄生振铃 |
| $I_{L,k}$ / $I_\Sigma$ | 单相电流、均流和纹波抵消 |
| $V_{OUT}$ | 负载阶跃下陷、过冲与恢复 |

**优点**：结构易理解；固定频率便于 EMI 与滤波设计；电流采样要求较低。  
**缺点**：外环直接面对 LC 双极点；负载跃变初期依赖输出电容；相间均流通常需专门的控制环。

## 5. 峰值电流模式（Peak Current Mode）

### 5.1 控制框图和斜率补偿

![峰值电流模式的独立放大框图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/02b_peak_current_mode_control.svg)

**图 3B.** 电流检测斜坡与补偿斜坡在比较器相加，比较器复位 PWM 锁存器；VOUT 反馈闭合较慢的外电压环。

Buck 电感电流上升斜率和下降斜率幅值：

$$m_1=\frac{V_{in}-V_o}{L},\qquad m_2=\frac{V_o}{L}$$

占空比 $D>0.5$ 时可能发生周期二分/次谐波振荡。常用斜率补偿稳定性起点：

$$m_a>\frac{m_2-m_1}{2}\quad(m_2>m_1)$$

斜率必须换算到比较器相同量纲；电流感测增益为 $R_s$ 时，比较器看到的是 $R_sm_1$ 和 $R_sm_2$。

### 5.2 低频小信号近似和节点波形

峰值模式是采样系统，精确模型需考虑采样效应。内电流环闭合、低频时，外电压环近似看到输出阻抗：

$$G_{vc}(s)\approx Z_o(s),\qquad T_{v,PCM}(s)\approx G_{cv}(s)H(s)Z_o(s)$$

比较器阈值以电流表示时，低频调制增益可作粗略估计：

$$\frac{\hat d}{\hat i_{cmd}}\approx\frac{1}{T_s(m_1+m_a)}$$

该式不是完整的采样数据模型，不能用来替代控制器高频模型。

图 4B 展示时钟置位、采样电流斜坡与阈值比较的相对关系。斜率补偿斜坡必须与采样电流换算到同一比较器量纲。

**优点**：快速逐周期限流；外电压环相对简单；容易实现逐相电流控制。  
**缺点**：斜率补偿不足会次谐波；采样尖峰/噪声会误触发；峰值和平均值不同，可能造成均流偏差。

## 6. 平均电流模式（Average Current Mode）

### 6.1 控制框图和传递函数

![平均电流模式的独立放大框图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/02c_average_current_mode_control.svg)

**图 3C.** 总电流命令经均流分配产生各相参考；每相的平均电流误差放大器控制 PWM。

若电流环带宽内输出电压可视为刚性电压，占空比到电感电流近似为：

$$G_{id}(s)=\frac{\hat i_L(s)}{\hat d(s)}\approx\frac{V_{in}}{sL+r_L}$$

电流环开环增益：

$$T_i(s)=G_{ci}(s)K_{PWM}G_{id}(s)K_{sense}$$

内环闭合后，外环可近似为：

$$T_{v,ACM}(s)\approx G_{cv}(s)H(s)G_{i,closed}(s)Z_o(s)$$

一种均流误差定义：

$$e_k=I_{avg}-I_{L,k}$$

内电流环带宽应高于电压环，但要远离开关采样、PWM 更新和 ADC 延迟限制区。

### 6.2 波形示意

图 4C 中 IAVG 贴近逐相 IREF 的低频分量，叠加开关频率电感纹波；实际均流误差取决于采样精度和均流环。

**优点**：直接约束平均电流，均流和电流监测精度好，适合高电流 VRM。  
**缺点**：电流采样精度、偏置和滤波影响大；环路更多，需处理电流环、电压环和均流环交互。

## 7. 恒定导通时间控制（CoT/COT）

### 7.1 原理和框图

反馈到达阈值时触发宽度近似固定的导通脉冲，关断后等待下一次触发。它不是固定频率锯齿波 PWM。

![恒定导通时间 CoT 的独立放大框图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/02d_cot_control.svg)

**图 3D.** 反馈纹波触发单稳态生成导通脉冲；输出经纹波路径回到比较器。实际信号注入方式依控制器实现而异。

理想稳态：

$$V_o\approx V_{in}\frac{T_{on}}{T_s},\qquad f_s\approx\frac{V_o}{V_{in}T_{on}}$$

自适应导通时间若以目标频率 $f_{target}$ 为目标，一个理想起始关系为：

$$T_{on}\approx\frac{V_o}{V_{in}f_{target}}$$

### 7.2 低 ESR 与波形

CoT 常依赖反馈纹波决定触发时刻。低 ESR 电容时纹波可能不足，可注入电感纹波、开关节点纹波或内部合成纹波。纹波幅度、相位和延迟影响触发抖动、开关频率和环路行为。

图 4D 中导通脉冲宽度近似固定，但相邻触发时刻会随反馈纹波和负载变化；曲线不代表固定频率运行。

CoT 是事件触发系统，通常没有覆盖所有工作点的统一 PWM 调制器增益。线性小信号模型只在特定稳态附近有效。需要检查固定/自适应 $T_{on}$、传播延迟、轻载跳脉冲、DCM、相间拍频和相位管理。不同厂商 CoT 变体差异很大。

**优点**：负载阶跃后响应快；控制器可集成化。  
**缺点**：频率可变；低 ESR 需纹波注入；EMI 和多相同步需按具体控制器验证。

## 8. 概念波特图与瞬态图

![Buck 环路增益与负载阶跃输出响应的定性对照图。](./Multi_phase_vr_sycnhronous_buck_loop_control_document_figures/05_loop_gain_and_transient.svg)

**图 5.** 上图用对数频率轴解释功率级斜率、补偿和交越频率；下图比较负载上升阶跃时的定性下陷与恢复。曲线是概念示意，不用于推断某一控制器必然更快或稳定。

此图只用于解释环路行为，不构成某种控制必然更快的结论；实际响应由输出电容、环路带宽、限流、控制器延迟、相数和负载边沿共同决定。

## 9. 四相 VR 数值示例

参数仅用于演示：$V_{in}=12\,V$，$V_o=1.2\,V$，$N=4$，每相 $f_s=500\,kHz$，$L=100\,nH$，$C=2\,mF$，$I_o=100\,A$，等效 ESR $0.5\,m\Omega$。

占空比、每相平均电流、相位差：

$$D\approx\frac{1.2}{12}=0.1,\qquad I_{L,avg}\approx\frac{100}{4}=25\,A$$

$$\Delta t_{phase}=\frac{1}{4\times500\,kHz}=0.5\,\mu s$$

每相电感纹波：

$$\Delta I_{L,pp}=\frac{(12-1.2)\times0.1}{100\,nH\times500\,kHz}\approx21.6\,A$$

$$I_{peak}\approx25+10.8=35.8\,A,\qquad I_{valley}\approx25-10.8=14.2\,A$$

由于 $D=0.1<1/N=0.25$，理想合成纹波：

$$\Delta I_{\Sigma,pp}\approx\frac{12}{100\,nH\times500\,kHz}\times0.1(1-4\times0.1)\approx14.4\,A$$

粗略电容纹波估算：

$$\Delta V_C\sim\frac{14.4}{8\times4\times500\,kHz\times2\,mF}\approx0.45\,mV$$

$$\Delta V_{ESR}\sim0.5\,m\Omega\times14.4\,A\approx7.2\,mV$$

该输出纹波估算未计入电容阵列 ESL/ESR 分布、布局寄生和非对称波形，应通过实际电容电流积分、开关仿真和示波器测量确认。

## 10. 设计与验证检查表

1. 确认控制模式、相数、每相频率、相位关系以及 CCM/DCM 范围。
2. 建立包含 $L$、$C$、ESR、DCR、负载和控制器延迟的模型。
3. 明确输出精度、负载阶跃、允许下陷/过冲、限流和均流目标。
4. 设计电压环、电流内环和均流环；CoT 按具体芯片建模。
5. 扫描输入、负载、温度和电容容差，验证相位/增益裕度与瞬态。
6. 同时观察单相电流和总电流，防止总电流正常但某相过载。

| 场景 | 观察节点/指标 |
|---|---|
| 满载稳态 | 各相平均/峰值电流、总纹波、输出纹波、开关频率 |
| 负载上升/下降 | VOUT 下陷/过冲、恢复时间、占空比、反向电流 |
| 输入阶跃 | 输入前馈、PWM/导通时间变化、输出扰动 |
| 相电流失配 | 均流环恢复速度和静态误差 |
| 轻载/跳相 | 频率、相位、低频纹波、DCM 切换 |
| 限流/故障 | 峰值/平均限流点、消隐时间、恢复逻辑 |

测输出纹波时使用短地弹簧或合适的差分探头；长接地线会引入虚假尖峰。电流探头需要校零。环路注入测量时，注入点不能破坏反馈直流偏置。

## 11. 选型建议

| 需求 | 优先考虑 | 核心验证项 |
|---|---|---|
| 固定频率、结构简单 | 电压型 | Type III 补偿和独立均流 |
| 快速逐周期限流 | 峰值电流模式 | 斜率补偿、噪声和峰均误差 |
| 高精度均流/电流监测 | 平均电流模式 | 采样精度、带宽分离、环间耦合 |
| 快负载响应、专用控制器 | CoT | 频率变化、纹波注入、EMI、多相同步 |

不能只按响应速度选控制方式；还要比较效率、热分布、均流精度、输出电容、EMI、故障响应和控制器可获得性。

## 12. 参考资料

1. R. B. Ridley, “A new, continuous-time model for current-mode control (power convertors),” *IEEE Transactions on Power Electronics*, vol. 6, no. 2, pp. 271–280, 1991. [DOI: 10.1109/63.76813](https://doi.org/10.1109/63.76813)
2. S. R. Sanders, J. M. Noworolski, X. Z. Liu, and G. C. Verghese, “Generalized averaging method for power conversion circuits,” *IEEE Transactions on Power Electronics*, vol. 6, no. 2, pp. 251–259, 1991. [DOI: 10.1109/63.76811](https://doi.org/10.1109/63.76811)
3. X. P. Xu, Y. C. Ren, Y. Mao, and F. C. Lee, “A family of novel interleaved DC/DC converters for low-voltage high-current voltage regulator module applications,” *IEEE PESC*, 2001, pp. 1507–1511. [DOI: 10.1109/PESC.2001.954332](https://doi.org/10.1109/PESC.2001.954332)
4. T. Senanayake and T. Ninomiya, “Multiphase voltage regulator module with current amplification and absorption technique,” *IEEE APEC*, 2004, pp. 1269–1274. [DOI: 10.1109/APEC.2004.1295986](https://doi.org/10.1109/APEC.2004.1295986)
5. J. M. García-Tenorio et al., “Input current characterization for interleaved multiphase Voltage Regulator Modules,” *IEEE IECON*, 2009, pp. 566–571. [DOI: 10.1109/IECON.2009.5414983](https://doi.org/10.1109/IECON.2009.5414983)
6. R. W. Erickson and D. Maksimović, *Fundamentals of Power Electronics*, 2nd ed., Springer. 标准教材，涵盖 PWM Buck 平均模型、补偿和电流模式控制基础。

> 文献范围说明：参考文献支撑电流模式模型、广义平均建模和多相 VRM 背景；正文公式是所列假设下的标准模型推导，并非声称每篇论文都逐式给出了这些表达式。CoT 控制器实现差异较大，最终设计应结合具体控制器数据手册、厂商参考设计、开关级仿真和实测。Nature/Science 通常不是这类电源控制电路设计文献的主要发表渠道。
