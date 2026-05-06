---
title: 全差分套筒式共源共栅结构 Cascode单级运放设计
date: 2026-05-06 22:30:00
categories:
  - 模拟IC
tags:
  - CMOS
  - OTA
  - Cascode
  - CMFB
  - gm/Id
cover: /img/cascode/cascode-schematic.png
---

这篇记录一次低压套筒式共源共栅全差分 OTA 的设计过程。电路工作在 $V_{DD}=1.1V$ 下，目标是开环差分小信号直流增益大于 $40dB$，单位增益带宽约 $500MHz$，负载电容 $C_L=1pF$。设计主线是：先用电压余量确定每层管子的偏置，再用 gm/Id 估算主放大器尺寸，最后加入 CMFB 并根据寄生电容迭代带宽。

![套筒式共源共栅 OTA 原理图与最终尺寸](/img/cascode/cascode-schematic.png)

参考资料：

- 王小桃，全差分单级运放设计：套筒式共源共栅结构 Cascode：[知乎专栏](https://zhuanlan.zhihu.com/p/1926361150477546875)
- 王小桃，共模反馈 CMFB：[知乎专栏](https://zhuanlan.zhihu.com/p/14156382741)
- 孙楠、刘佳欣、揭路：《现代模拟集成电路设计》

## 拓扑与指标

主放大器采用 NMOS 输入差分对，PMOS 共源共栅有源负载，上下各堆叠多只 MOS 管以提高输出电阻和直流增益。由于是全差分输出，电路必须配套共模反馈 CMFB，否则 $V_{op}$、$V_{on}$ 的平均值会由器件失配和电流误差漂移。

| 指标 | 目标 / 设计值 |
| --- | ---: |
| 电源电压 | 1.1 V |
| 单管预留 $V_{DS}$ | 约 200 mV |
| 开环增益 | $>40dB$ |
| GBW | 500 MHz |
| 负载电容 | 1 pF |

套筒结构的核心矛盾是增益和电压余量。堆叠管越多，输出电阻越高，增益越容易做上去；但在 1.1 V 电源下，每只管子的 $V_{DS}$ 都必须压得很紧，否则输入、输出或共模范围马上不够。

## 电压余量与偏置

手算中先取每只管子约 $V_{DS}=200mV$。若按 $V_{GS}\approx V_{DSAT}+V_{TH}\approx 500mV$ 估计，则几组关键偏置可以从堆叠电压直接得到：

$$
V_{BN}\approx V_{GS}\approx0.5V
$$

$$
V_{BIAS}=V_{DD}-V_{GS}\approx0.6V
$$

$$
V_{CM}\approx V_{GS}+V_{DS}\approx0.7V
$$

$$
V_{BB1}=V_{DD}-V_{DS}-V_{GS}\approx0.4V
$$

$$
V_{BB2}=2V_{DS}+V_{GS}\approx0.9V
$$

$V_{B3}$ 用来卡住输入管和 PMOS 共源共栅管之间的工作点。$V_{B3}$ 过低时，输入对 $M1/M2$ 会进入线性区；$V_{B3}$ 过高时，上方 $M3/M4$ 会进入线性区。笔记里采用的思路是令 $V_{B3}$ 与输入共模附近的内部节点相匹配，再通过仿真微调。

## 主放大器尺寸

开环增益可近似写成：

$$
A_v\approx
g_{m1}
\left[
(g_{m2}r_{o2}r_{o1})
\parallel
(g_{m3}r_{o3}r_{o4})
\right]
\approx \frac{1}{2}g_m^2r_o^2
$$

为了让增益超过 $40dB$，即电压增益超过 100，手算中取：

$$
g_mr_o\gtrsim20
$$

带宽由输入级跨导和负载电容主导：

$$
GBW\approx\frac{g_{m1}}{2\pi C_L}
$$

代入 $GBW=500MHz$、$C_L=1pF$：

$$
g_{m1}\approx2\pi\cdot500MHz\cdot1pF\approx3.14mS
$$

输入对取 $g_m/I_D=15V^{-1}$，则：

$$
I_D\approx\frac{3.14mS}{15V^{-1}}\approx210uA
$$

查表后得到的第一轮尺寸大致如下，随后再根据 PVT、寄生和 CMFB 稳定性微调。

| 器件 | 作用 | 设计依据 | 初始尺寸量级 |
| --- | --- | --- | ---: |
| M1/M2 | NMOS 输入对 | $g_m\approx3.14mS,\ g_m/I_D=15$ | $W/L\approx9.5u/0.1u$ |
| M3/M4 | PMOS 共源共栅管 | 保证 $g_mr_o$ 与电压余量 | $W/L\approx43.5u/0.15u$ |
| M5/M6/M7/M8 | PMOS 电流源与上方堆叠 | 复制上支路电流并提高 $r_o$ | 见最终原理图 |
| M9 | NMOS 尾电流源 | $I_{tail}\approx420uA$ | $W/L\approx73u/0.4u$ |

## CMFB 设计

全差分 OTA 的差模环路只决定 $V_{op}-V_{on}$，并不决定输出共模：

$$
V_{OC}=\frac{V_{op}+V_{on}}{2}
$$

因此需要 CMFB 把 $V_{OC}$ 拉回 $V_{REF}$。笔记中先分析了连续时间 CMFB 和开关电容 CMFB 的区别：CT-CMFB 连续工作，但会把检测网络的电阻、电容带入环路；SC-CMFB 适合离散时间系统，但需要时钟。

最初的电阻式检测可以近似得到：

$$
V_m\approx\frac{V_{op}+V_{on}}{2}
$$

但有限输出电阻会引入额外的 $R_{cm}$ 与 $C_{cm}$，对应零点和极点：

$$
\omega_z\approx\frac{1}{R_{cm}C_{cm}}
$$

$$
\omega_p\approx
\frac{1}{R_{cm}(C_{cm}+C_{in,EA}/2)}
$$

为了避免检测网络显著污染主放大器节点，最终采用晶体管式 CMFB，通过 $V_{CTRL}$ 调整上方 PMOS 电流源。直观上，如果输出共模偏高，CMFB 会改变 $V_{CTRL}$，让上支路电流减小或下拉电流相对增强，使输出共模回落；反之亦然。

## 寄生与环路迭代

主放大器的输出节点电阻很大，$C_L$ 和寄生电容会直接压低高频带宽。手算中曾按理想 $C_L=1pF$ 推出 $g_{m1}$，但加入器件寄生后，原始 GBW 约为 412 MHz，低于 500 MHz 目标。

笔记中用一个简单系数做迭代：

$$
k=\frac{1}{2-GBW_{spec}/GBW_{raw}}
$$

代入：

$$
k=\frac{1}{2-500M/412M}\approx1.28
$$

这意味着需要把相关跨导、电流和尺寸再往上推一轮，才能抵消寄生电容带来的带宽损失。这个步骤也说明：高速 OTA 的手算只能给出第一版，最终一定要靠含寄生的仿真闭环。

## 仿真结果

最终仿真指标如下：

![Cascode OTA 仿真指标汇总](/img/cascode/cascode-sim-summary.png)

| 指标 | 仿真值 |
| --- | ---: |
| 功耗 | 997.1 uW |
| GBW | 525.2 MHz |
| 开环增益 | 40.03 dB |
| PM | 89.5 deg |

输出范围仿真如下图。以约 $40dB$ 的峰值增益为参考，下降到 $37dB$ 的边界约在 $\pm340mV$，因此可用输出摆幅约为 $680mV_{pp}$。

![Cascode OTA 输出范围仿真](/img/cascode/cascode-output-range.png)

## 小结

这版套筒式共源共栅 OTA 的设计关键在三个地方：第一，低压下先分配 $V_{DS}$ 和偏置电压，确保每层管子仍在饱和区；第二，用 gm/Id 把 $GBW$ 和增益要求转换成输入对电流与尺寸；第三，全差分输出必须认真做 CMFB，并把 CMFB 环路和主差模环路分开验证。

最终结果为 $GBW=525.2MHz$、$A_v=40.03dB$、$PM=89.5^\circ$，基本达到 500 MHz / 40 dB 的设计目标。
