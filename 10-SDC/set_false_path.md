---
title: set_false_path
aliases: [False Path, 假路径约束]
tags: [SDC, STA, PrimeTime, FusionCompiler, 时序例外]
created: 2026-09-15
---

# set_false_path

> 将指定路径的时序检查标记为不适用。连接和物理延迟仍存在；这条命令不会证明功能正确，也不会解决 CDC 亚稳态问题。

## 详细教程

[打开 set_false_path 中文详细教程](<file:///C:/Users/32549/Desktop/obsidian/fly/fly/10-SDC/set_false_path.html>)

[同目录相对链接（便于随笔记迁移）](set_false_path.html)



## 关键语法

```tcl
# 独立教学示例；替换为真实合法对象后使用
set_false_path -from [get_pins u_src/CK] \
    -through [get_pins u_mux/A1] \
    -to [get_pins u_dst/D]

# 单向 A → B；不包含 B → A
set_false_path -from [get_clocks CLK_A] -to [get_clocks CLK_B]
```

- `-from`：发射起点；如输入 port、寄存器时钟 pin、合法 sequential cell，或 launch clock。
- `-to`：捕获终点；如输出 port、寄存器数据 pin、合法 sequential cell，或 capture clock。
- `-through`：路径必须经过的对象；同一列表是 OR，重复使用是按先后顺序的 AND。FC 本文版本支持 pin/port/cell/net；优先明确 pin。
- 省略 `-setup/-hold` 默认两类都切；只写一类则只切该类。`-rise/-fall` 按终点数据转换限定。
- `get_clocks` 与 `get_ports` 不等价；寄存器时钟端 CK/CP 是常用规范起点，Q 通常用于 through。

## PT / FC 检查速查

| 检查任务 | PrimeTime | Fusion Compiler T-2022.03-SP4 |
| --- | --- | --- |
| 查看例外 | `report_exceptions`、`report_exceptions -ignored` | 同名命令，另可用 `report_exceptions -dominant` |
| 路径主导例外 | `report_timing … -exceptions dominant` | `report_timing … -exception dominant` |
| 主导与被覆盖例外 | `report_timing … -exceptions all` | 本版不提供此 `all` 用法；结合 `report_exceptions -ignored` |
| 允许报告未约束路径 | `set_app_var timing_report_unconstrained_paths true` | `set_app_options -name time.report_unconstrained_paths -value true` |
| 时钟组 | `report_clock -groups` | `report_clock -groups` |

报告选项按工具和版本区分，先查 `man report_timing`。调试结束后恢复未约束路径报告的原设置；完整保存/恢复示例见 HTML。

## 重点注意

1. **普通报告看不到不等于验证成功。** 完全被 false 的检查通常不再出现在普通 setup/hold 报告中；也可能因 case analysis、禁用弧、无时钟或筛选条件而消失。
2. `-exceptions/-exception` 为已报出的路径显示例外，不保证枚举所有 false path。先核对集合、基线、例外与时钟组，再检查应豁免和应保留的样本。
3. 不要用大范围通配符隐藏真实违例。`-to` 单独使用可能切掉所有到达该终点的合法路径；同步器第一级到第二级应保留同域检查。
4. **异步 clock groups 不等于两条 false path 的完全替换。** 常规 `-asynchronous` 表达组间双向关系，还影响 SI 窗口。明确列入派生时钟；只有一个 group 可能把它与所有其他时钟隔离。
5. **多周期仍需满足时序。** `set_multicycle_path` 适用于功能允许多拍捕获；setup/hold 边沿需推导，不能用它把真实异步时钟变成同步时钟。
6. 同一路径的例外会按类型及具体范围竞争；不要假设后写的一定生效。重叠 false path 可能使 max/min delay 或 multicycle 失效。
7. FC 本文版本的 `set_false_path`、`report_exceptions` 针对当前 mode；`report_timing` 可跨场景。FUNC / SCAN_SHIFT / SCAN_CAPTURE 分别验证，检查报告所属场景。
8. 异步复位涉及 recovery/removal；不要无依据整网豁免。`set_false_path -reset_path` 也不是简单“取消 false”。

## 相关知识

- [[set_clock_groups]]：异步 / 互斥时钟关系
- [[set_multicycle_path]]：多周期与 setup/hold 边沿
- [[report_timing]]、[[report_exceptions]]：路径与例外检查
- [[set_max_delay]]、[[set_min_delay]]：明确延迟预算
- [[set_case_analysis]]、[[set_disable_timing]]：模式选择与禁用时序弧
- [[create_clock]]、[[create_generated_clock]]：时钟与派生关系
- [[CDC]]、[[recovery_removal]]：跨域与复位释放

相关双链是知识节点入口，尚未创建的同名笔记可日后补充。

