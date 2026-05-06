---
title: 基本 CMOS 二级密勒补偿 OTA 设计
date: 2026-05-06 22:00:00
categories:
  - 模拟IC
tags:
  - CMOS
  - OTA
  - gm/Id
  - 密勒补偿
cover: /img/ota/ota-cadence-sizing.png
---

这篇记录一次基本二级 CMOS OTA 的手算与仿真闭环。电路由四个部分组成：左侧是自偏置电流基准，中间是 PMOS 输入差分对和 NMOS 电流镜负载，右侧是 NMOS 共源第二级与 PMOS 电流源负载，输出端挂载 $C_L$，并用 $M14-C_C$ 串联网络做密勒补偿。

![基本 CMOS 二级密勒补偿 OTA 原理图](/img/ota/ota-schematic-original.JPEG)

参考资料：

- gm/Id 法 B 站视频：[BV1PZ4y1g7cZ](https://www.bilibili.com/video/BV1PZ4y1g7cZ/)
- gm/Id 法文字说明：[知乎专栏](https://zhuanlan.zhihu.com/p/1899593665992696280)
- Silveira, Flandre, Jespers, [A gm/ID based methodology for the design of CMOS analog circuits](https://www.researchgate.net/publication/2977303_A_gmID_based_methodology_for_the_design_of_CMOS_analog_circuits_and_its_application_to_the_synthesis_of_a_silicon-on-insulator_micropower_OTA)
- Murmann gm/Id 设计流程整理：[Book-on-gm-ID-design / Design Flow](https://deepwiki.com/bmurmann/Book-on-gm-ID-design/3.1-design-flow)
- 两级运放补偿综述参考：[Two-Stage Operational Amplifier Design by Using Direct and Indirect Feedback Compensations](https://vtechworks.lib.vt.edu/handle/10919/103938)

## gm/Id 法的核心

gm/Id 也叫跨导效率，它表示单位偏置电流能换来多少跨导：

$$
\frac{g_m}{I_D}=\frac{\partial I_D / \partial V_{GS}}{I_D}
=\frac{\partial \ln I_D}{\partial V_{GS}}
$$

这个量比直接用 $V_{OV}$ 更适合现代 CMOS，因为它能把弱反型、中等反型、强反型放在同一套曲线上看。设计时先由系统指标推出每只管子需要的 $g_m$ 和 $I_D$，再在工艺仿真的查找表中按 $g_m/I_D$、$L$ 查询电流密度 $I_D/W$，最后得到宽度：

$$
I_D=\frac{g_m}{g_m/I_D},\qquad
W=\frac{I_D}{I_D/W}
$$

直觉上，$g_m/I_D$ 越大，同样电流下跨导越高，功耗效率好，但速度、摆幅和匹配会受限；$g_m/I_D$ 越小，管子更偏强反型，速度和摆幅余量更好，但要花更多电流。

## 电路拓扑识别

第一级是 PMOS 输入差分对 $M1/M2$。尾电流源 $M5$ 从 VDD 向源节点供电，$M3/M4$ 是 NMOS 电流镜有源负载，单端输出取在节点 3，也就是 $M2$ 与 $M4$ 的漏端。

第二级是共源放大器。$M6$ 是 NMOS 放大管，栅极接节点 3；$M7$ 是 PMOS 电流源负载，输出节点为 $VOUT$。该级负责把第一级的电压摆动进一步放大并驱动 $C_L$。

补偿网络由工作在线性区的 $M14$ 和 $C_C$ 串联而成。$M14$ 等效为零点调节电阻 $R_z$，用于处理普通密勒电容带来的右半平面零点。左侧偏置网络用 $M12/M13/R_B$ 建立参考电流，再通过 $M8/M9/M5/M7$ 等电流镜复制到各支路。

## 设计指标

本次目标来自手算笔记：

| 参数 | 目标 |
| --- | ---: |
| $V_{DD}$ | 1.2 V |
| GBW | 100 MHz |
| $C_L$ | 3 pF |
| $C_C$ | 1 pF |
| 目标非主极点 | $f_{p2}\approx 3\times GBW$ |

单位增益带宽主要由第一级输入跨导和补偿电容决定：

$$
GBW \approx \frac{g_{m1}}{2\pi C_C}
$$

所以：

$$
g_{m1}\approx 2\pi\cdot100MHz\cdot1pF=0.628mS
$$

为了让非主极点离单位增益频率有足够距离，取：

$$
f_{p2}\approx 3GBW=300MHz
$$

第二级近似由 $M6$ 的跨导推动负载电容：

$$
g_{m6}\approx 2\pi f_{p2}C_L
=2\pi\cdot300MHz\cdot3pF
\approx5.65mS
$$

这一步也解释了为什么二级 OTA 往往需要明显大于第一级的第二级电流：如果不提高 $g_{m6}$，非主极点会跟着压到 GBW 附近，相位裕度马上变差。

## 参数设计

按 gm/Id 查表得到的首轮尺寸如下。这里 $M1/M2$ 取较高的 $g_m/I_D=15V^{-1}$，用于提高输入级跨导效率；电流镜、尾电流源和第二级多取 $10V^{-1}$，折中速度、摆幅和电流。

| 器件 | 作用 | 类型 | $g_m/I_D$ | $g_m$ | $I_D$ | $L$ | $W$ |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: |
| M1/M2 | 第一级输入对 | PMOS | 15 | 0.6 mS | 0.04 mA | 0.5 um | 30.5 um |
| M3/M4 | 第一级镜像负载 | NMOS | 10 | 0.4 mS | 0.04 mA | 0.5 um | 3.1 um |
| M5 | 第一级电流源 | PMOS | 10 | 0.8 mS | 0.08 mA | 0.5 um | 20.4 um |
| M6 | 第二级共源管 | NMOS | 10 | 5.7 mS | 0.57 mA | 60 nm | 7.2 um |
| M7 | 第二级电流源 | PMOS | 10 | 5.7 mS | 0.57 mA | 0.5 um | 145.4 um |

![Cadence 中的最终尺寸与连接](/img/ota/ota-cadence-sizing.png)

查表时用到的电流密度关系大致为：

| 管型与沟长 | $g_m/I_D=10$ | $g_m/I_D=15$ |
| --- | ---: | ---: |
| NMOS, $L=0.5um$ | $I_D/W\approx13.03uA/um$ | $4.24uA/um$ |
| NMOS, $L=60nm$ | $I_D/W\approx78.7uA/um$ | - |
| PMOS, $L=0.5um$ | $I_D/W\approx3.92uA/um$ | $1.31uA/um$ |

例如 $M1/M2$ 的电流为：

$$
I_D=\frac{0.6mS}{15V^{-1}}\approx40uA
$$

PMOS 在 $L=0.5um$、$g_m/I_D=15$ 时 $I_D/W\approx1.31uA/um$，所以：

$$
W\approx\frac{40uA}{1.31uA/um}\approx30.5um
$$

$M6$ 为了把非主极点推到约 300 MHz，电流约为：

$$
I_D=\frac{5.65mS}{10V^{-1}}\approx0.565mA
$$

在 $L=60nm$、$g_m/I_D=10$ 的 NMOS 电流密度约 $78.7uA/um$，所以：

$$
W\approx\frac{565uA}{78.7uA/um}\approx7.2um
$$

## 偏置电路

偏置部分近似为 beta-multiplier reference。手算中取：

$$
\frac{(W/L)_{12}}{(W/L)_{13}}=4
$$

按长沟道平方律，参考电流可写成：

$$
I_B=
\frac{2}{\mu_n C_{ox}(W/L)_{12}R_B^2}
\left(
\sqrt{\frac{(W/L)_{12}}{(W/L)_{13}}}-1
\right)^2
$$

当宽长比之比为 4 时：

$$
g_{m12}=\frac{2}{R_B},\qquad g_{m13}=\frac{1}{R_B}
$$

若希望 $g_{m12}\approx0.8mS$，则手算初值：

$$
R_B\approx\frac{2}{0.8mS}=2.5k\Omega
$$

实际 BSIM 模型下，平方律只是初始估算。最终示意图中 $R_B$ 调到约 $1.3k\Omega$，偏置支路电流约 $80uA$，再由 PMOS 电流镜复制给 $M5$ 和 $M7$。偏置镜像管采用较长 $L=1um$，是为了减小沟道长度调制，提高电流复制精度。

![Cadence 工作点标注图](/img/ota/ota-cadence-operating-point.png)

## 补偿网络

没有串联电阻时，普通密勒电容会引入右半平面零点：

$$
\omega_z\approx\frac{g_{m6}}{C_C}
$$

右半平面零点会额外拉低相位。串联 $R_z$ 后，零点近似为：

$$
\omega_z\approx
\frac{1}{C_C(R_z-1/g_{m6})}
$$

当：

$$
R_z\approx\frac{1}{g_{m6}}\approx\frac{1}{5.7mS}\approx175\Omega
$$

零点被推到很高频；若 $R_z$ 略大于 $1/g_{m6}$，零点进入左半平面，可用于补偿非主极点的一部分相位损失。本电路中 $M14$ 工作在线性区，等效实现这个小电阻。

## 设计结果

仿真结果如下：

![仿真指标汇总](/img/ota/ota-sim-results.png)

| 指标 | 仿真值 |
| --- | ---: |
| 功耗 | 991.7 uW |
| GBW | 102.4 MHz |
| 相位裕度 PM | 61.84 deg |
| 开环增益 | 47.34 dB |
| CMRR | 67.5 dB |
| PSRR | 43.66 dB |

GBW 和 PM 基本达到目标。开环增益不算高，主要受短沟道第二级 $M6$ 和电流源有限输出电阻影响；若要提高增益，可以把第二级或电流源沟长加长，或者改用 cascode / gain-boosting，但代价是输出摆幅、速度和电压余量。

## 遗留问题：为什么不把 Cc 取得极小？

直观看，$GBW\approx g_{m1}/(2\pi C_C)$，减小 $C_C$ 的确会把 GBW 往上推；再加一个合适的 $R_z$，似乎还能把相位裕度救回来。所以问题是：为什么经验上还常限制 $C_C\approx0.2\sim0.5C_L$，而不是把 $C_C$ 取得极小？

关键在于 $C_C$ 不只决定 GBW，也决定极点分裂的强弱。二级 OTA 的非主极点近似为：

$$
\omega_{p2}\approx\frac{g_{m6}}{C_L}
$$

单位增益角频率为：

$$
\omega_u\approx\frac{g_{m1}}{C_C}
$$

两者的距离为：

$$
\frac{\omega_{p2}}{\omega_u}
\approx
\frac{g_{m6}}{g_{m1}}\frac{C_C}{C_L}
$$

本设计中 $g_{m6}/g_{m1}\approx5.65/0.63\approx9$，$C_C/C_L=1/3$，所以：

$$
\frac{\omega_{p2}}{\omega_u}\approx3
$$

这正好对应手算里“非主极点约为 $3\times GBW$”的稳定性余量。若把 $C_C$ 改成 $0.05C_L$，在 $g_{m6}/g_{m1}$ 不变时：

$$
\frac{\omega_{p2}}{\omega_u}\approx9\times0.05=0.45
$$

也就是说，单位增益频率已经跑到非主极点之后了，闭环会严重欠阻尼甚至不稳定。要保持同样的极点距离，就必须把 $g_{m6}/g_{m1}$ 提到大约 60，这会直接带来第二级电流、面积和寄生电容的暴涨。

$R_z$ 能处理零点，甚至可以用左半平面零点抵消一部分非主极点相位滞后，但它不是免费的万能补偿。原因有三点：

1. 极点-零点精确抵消对 PVT、$C_L$、偏置电流和 MOS 电阻非线性非常敏感，仿真标称角上能抵消，不代表全角全负载都能抵消。
2. $C_C$ 太小时，它会接近版图寄生电容、节点寄生电容和模型误差量级，手算中的“可控补偿电容”会变成“不确定寄生网络的一部分”。
3. 更高的 GBW 会撞上器件 $f_T$、内部节点极点、输出节点寄生、电源/共模路径等高阶效应，二阶近似不再可靠。

所以 $C_C=0.2\sim0.5C_L$ 本质上不是数学定理，而是一条工程上好用的稳健性约束：用可接受的 GBW 换足够的极点分裂、负载容差、PVT 余量和可预测的瞬态响应。想继续减小 $C_C$ 可以，但必须同步提高 $g_{m6}$、重做全角稳定性仿真，并确认寄生电容已经纳入模型。
