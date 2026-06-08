# Embedded Hardware Bridge

把原理图变成固件的 Claude Code Skill。给它原理图（网表、PDF 或截图），
它出硬件事实表、生成 `board_pin.h` / `board_init.c` 脚手架，
然后交叉验证固件和电路是否一致。

## 解决什么问题

嵌入式程序员把大量时间耗在这些又繁琐又容易出错的事上：

- 对着原理图一个引脚一个引脚抄到代码里
- 手算 ADC 分压公式
- 跨多页原理图追 I2C/SPI/UART 连线
- 找那些"编译通过、烧进去炸板"的高低有效信号写反的 bug
- 排查寄存器位写错——代码逻辑全对，就是外设不干活

AI 编程工具（Claude Code、Copilot、Cursor）写应用逻辑很强，
但硬件正确性是它们的盲区——读不懂原理图、猜寄存器名、
不知道这个信号是高有效还是低有效。

**这个 Skill 给 AI 提供硬件事实，让它不再瞎猜。**

## 工作流

```
原理图（网表 / PDF / 截图）
         │
         ▼
    第1步：出硬件事实表
    - 引脚分配表
    - ADC 通道 & 分压公式
    - 通信总线拓扑
    - 信号极性标注
         │
         ▼
    第2步：生成固件脚手架
    - board_pin.h, board_init.c
    - 驱动骨架（ADC, PWM, I2C, GPIO）
         │
         ▼
    第3步：交叉验证
    - 逐寄存器核对
    - 硬件一致性检查清单
    - 烧录前安全底线
         │
         ▼
    第4步：交付
    - 编码转换、目录摊平
    - 调试观察变量
    - 联调测试方案
```

## 安装

```bash
git clone https://github.com/moziiiii/embedded-hardware-bridge.git \
  ~/.claude/skills/embedded-hardware-bridge
```

## 使用方法

### 有网表（首选——100% 准确）

先找硬件工程师导出网表，PADS 菜单里点一下的事：

```
[粘贴网表内容]

从这个网表提取完整的硬件事实表。MCU 型号：[填]。产品功能：[填]。
```

### 只有原理图截图/PDF（备选）

```
[附原理图]

提取硬件事实表。
MCU：SC8F096AD824NPR。
产品：口罩控制器，带风扇、电池、升压。
关键模块：按键×3、LED×4、TH06 温湿度传感器、HP203N 气压传感器、
CN3302 充电芯片、风扇 PWM、升压使能。
```

### 验证已有固件

```
对照硬件事实表检查这些文件：
- board_pin.h / board_init.c
- drv_adc.c / drv_pwm.c / drv_i2c.c

每个不一致的地方标出文件名、行号、正确值。
```

## 抓过的实际 Bug

| Bug | 怎么抓到的 |
|-----|-----------|
| TRIS 方向写反（输出设成输入） | 交叉验证 |
| 海拔/温度被除了两次 100 | 跨文件数据流追踪 |
| PWM 周期寄存器用了 TMR2 而不是 PWMTL | 寄存器终检 |
| 传感器 Init 调了但 500ms 任务里没调 Read | Init/Read 配对检查 |
| 充电态退出时残留 keyEvent 导致误开机 | 状态切换事件清零检查 |
| IDE 编码 UTF-8 → GBK 注释乱码 | 交付步骤 |
| 调试断点导致看门狗复位 | 烧录前检查清单 |

## 局限性

- **不能**解析私有 EDA 二进制格式（PADS `.sch`、Altium `.SchDoc` 等）。
  **永远优先要网表。**
- 从截图提取的引脚分配准确率约 95%，需人工复核。
- 不替代硬件调试（示波器、逻辑分析仪、JTAG）。它减少软件 bug，不修 PCB 错误。

## 许可

MIT

## 相关项目

- [Tansuo2021/ADtoKeil](https://github.com/Tansuo2021/ADtoKeil) — Altium `.SchDoc` → Keil 固件 Skill 套件
- [github/spec-kit](https://github.com/github/spec-kit) — 规范驱动开发框架
- [anthropics/skills](https://github.com/anthropics/skills) — Agent Skills 协议
