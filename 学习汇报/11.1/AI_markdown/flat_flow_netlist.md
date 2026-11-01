# flat_flow_netlist 实验流程与命令详解

实验目录：[flat_flow_netlist](E:/tessent/Reference_2026/tessent/flat_flow_netlist)，主要工作目录为 `flow`。

本文沿用已确认的 f_f_n 说明结构，并根据 E 盘完整材料重新核对 5 个脚本、5 份日志、原始网表、工艺/存储器模型、TSDB 和向量产物。下文“本目录”均指这个实验，而非 Markdown 保存目录。脚本说明的是“当前计划执行什么”，日志和向量说明的是“历史实际执行了什么”；两者不同的地方单独列出。

日志环境为 **Tessent Shell 2021.3、x86-64 Linux，历史运行日期为 2022-06-27**。本次是静态阅读和交叉核对，没有重新运行 EDA 工具。本次整理时间为 2026-10-30；E 盘源码保持原样。

## 1. 这个实验要完成什么

实验从已有的门级网表出发，为顶层模块 `top` 加入可测试性结构（DFT），然后生成 stuck-at（固定为 0 或固定为 1）故障测试向量。

```text
原始门级网表 top_netlist.v
        │
        ▼
1_insert_pad：加入输入/输出 PAD，预留测试端口
        │  ./netlist/top.v
        ▼
2_insert_bscan&mbist：加入 TAP、边界扫描、IJTAG、存储器自测试
        │  TSDB 中的 design_id=bscan
        ▼
3_insert_edt_occ：加入扫描压缩、片上时钟控制及测试控制信号
        │  TSDB 中的 design_id=edt
        ▼
4_insert_scan：分配并连接 6 条内部扫描链
        │  TSDB 中的 design_id=scan
        ▼
5_atpg：读取扫描设计，生成故障测试向量、统计结果并导出
```

几个结构的分工如下。关于边界扫描、IJTAG 与测试仪器的集成，可对照 [Siemens BoundaryScan 官方介绍](https://www.siemens.com/en-gb/products/ic/tessent/test/boundaryscan/)。

| 结构 | 本实验中的用途 |
|---|---|
| PAD | 在顶层逻辑端口和芯片外部端口之间加入 I/O 单元模型，为后续芯片级边界扫描提供接口结构 |
| Boundary Scan / BSCAN | 在芯片 I/O 边界加入可控制、可观察的扫描结构；本实验还通过辅助端口复用部分功能 I/O 作为 EDT 通道 |
| TAP | JTAG 测试访问控制器，通过 TCK/TMS/TDI/TDO/TRST 访问测试网络 |
| IJTAG | 连接和配置片内测试仪器；本实验用 SIB 选择访问分支，用 TDR 保存静态测试控制值 |
| MBIST | 为存储器生成片内自测试控制结构；历史配置中有 1 个控制器，连接 3 个存储器实例 |
| EDT | Embedded Deterministic Test：使用较少的外部通道装载较多内部扫描链，并压缩输出响应 |
| OCC | On-Chip Clock Controller：在测试模式下控制扫描/捕获时钟；本实验分别处理 clk、ramclk |
| Scan Chain | 把可扫描时序单元及已有扫描段连接成移位路径，便于装入状态并移出响应 |
| ATPG | 根据故障模型自动生成激励，结合故障仿真判断哪些故障被检测 |

**这里生成的是 stuck-at ATPG 向量。加入了 OCC，并不代表本目录已执行 transition 或 at-speed 故障测试。** 当前脚本没有切换到 transition 故障模型，日志标题和 STIL 头部都确认是 stuck-at。

## 2. 目录文件与实验数据

| 当前文件 | 应如何阅读 |
|---|---|
| [1_insert_pad](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/1_insert_pad) | 第一步的 Tcl/Tessent 命令脚本 |
| [2_insert_bscan&mbist](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/2_insert_bscan&mbist) | 第二步；文件名含 `&`，在操作系统命令行中必须正确加引号 |
| [3_insert_edt_occ](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/3_insert_edt_occ) | 第三步；主要参数集中在 DFT Specification 配置中 |
| [4_insert_scan](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/4_insert_scan) | 第四步；扫描模式创建、链分配和实际插入 |
| [5_atpg](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/5_atpg) | 第五步；ATPG 和向量导出 |
| 5 个同名 `.log` 文件 | 历史命令及执行结果；`command` 是会话命令，`sub-command` 往往是命令内部展开的操作 |
| [.synopsys_dc.setup](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/.synopsys_dc.setup) | Synopsys Design Compiler 的库配置，由综合流程使用 |
| [pattern.stil](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/pattern.stil) | 已保存的测试向量，包含信号、时序、移位宏、JTAG 初始化和测试数据 |

E 盘材料比之前的 f_f_n 归档完整，以下输入与历史输出均实际存在：

| 路径 | 当前内容 |
|---|---|
| `lib/` | adk.db、adk.lib、adk.tcelllib、adk.v、picdram 的测试描述/DFT/仿真模型 |
| `netlist/top_netlist.v` | 脚本实际读入的原始网表，文件内包含 adder 和 top |
| `flow/netlist/top.v` | 插 PAD 后网表 |
| `flow/tsdb/` | bscan、edt、scan 设计及仪器、ATPG 数据 |
| `flow/synthesis_outdir/` | 两次测试仪器综合脚本和产物 |
| `flow/patterns/` | STIL、串行和并行 Verilog 测试平台及配套数据 |
| `flow/simulation_outdir/` | ICL/BSCAN/MBIST 生成测试平台及数据；未找到运行 transcript |

本实验已有较完整的结构和向量产物，但原脚本的 Linux 工具安装路径、`.synopsys_dc.setup` 绝对路径和许可证仍需迁移。文件存在也不等于每个仿真都已经通过。

### 2.1 日志能确认的设计规模

| 项目 | 记录或计算依据 |
|---|---|
| 顶层模块 | `top` |
| 原始输入 | `net_in[15:0]`、`clk`、`ramclk`、`address_1[6:0]`、`address_2[6:0]`、`we`，共 33 个标量位 |
| 原始输出 | `net_out[7:0]`，共 8 位 |
| 新增输入 | TDI、TMS、TRST、TCK、scan_en、edt_update、test_clock，共 7 个 |
| 新增输出 | TDO，共 1 个 |
| 插 PAD 后接口 | 40 个输入、9 个输出；与后续日志的 PIs/POs 一致 |
| 存储器 | `dram_1`、`dram_2`、`dram_3`，由 MBIST 配置报告确认 |

原始 `top_netlist.v` 现可确认：设计包含 3 个 picdram、2 个 adder 实例和 16 个显式 dff。存储器模型是 70 个 8 位字，地址总线 7 位不意味着有效深度就是 128。`picdram.tcd_memory` 标注 SRAM、1RW、sync、SMarchCHKB，地址计数范围为 0～69。

## 3. 先区分三层命令

### 3.1 操作系统 Shell：启动工具和准备目录

```bash
/workspace/tessent/tessent_2021_3/bin/tessent -shell -log 1_insert_pad.log -replace
```

这是第一个日志头部记录的启动命令：

| 参数 | 作用 |
|---|---|
| `/workspace/.../bin/tessent` | 历史 Linux 环境中的 Tessent 可执行程序路径，换机器需换为实际安装路径 |
| `-shell` | 启动命令行交互界面 |
| `-log 文件名` | 将会话命令和工具输出记录到指定日志 |
| `-replace` | 允许替换已有日志；在不同命令中，它针对的是该命令要写出的文件 |

其余日志也使用 `tessent -shell -log ...` 启动。脚本没有扩展名，但内容是 Tcl/Tessent 命令，不能当作普通 Bash 脚本逐行执行。

### 3.2 Tessent 命令：加载、分析和修改设计

例如 `read_verilog`、`check_design_rules`、`process_dft_specification`、`insert_test_logic`、`create_patterns`。

### 3.3 Tcl 语法：组织命令和处理对象

| 写法 | 本实验中的作用 |
|---|---|
| `set top top` | 建立变量 `top`，其值为字符串 `top` |
| `$top`、`${top}` | 引用变量，例如 `${top}.v` 展开为 `top.v` |
| `[命令]` | 执行内部命令，把返回值作为外层命令参数 |
| `{ ... }` | 保存命令块、列表或配置文本；如 `read_config_data -from_string { ... }` 把内容交给配置解析器 |
| `foreach` | 遍历普通 Tcl 列表 |
| `foreach_in_collection` | 遍历 Tessent 对象集合，例如端口集合 |
| `list` | 构造 Tcl 列表 |
| `lindex 列表 0` | 取列表的第一个元素 |
| `string map` | 按映射替换字符串字符，本实验用于清理实例名中的 `[` 和 `]` |
| 行末 `\` | 将下一行拼接到当前命令 |
| `#` | Tcl 脚本中的注释，不执行 |

`read_config_data` 的大括号内部是 Tessent 配置语法，例如 `Controller(c1) { ... }`、`key : value;`，不是逐条 Tcl 命令。

## 4. 五步共同使用的基础命令

| 命令 | 含义与本实验中的位置 |
|---|---|
| `set_context dft -no_rtl` | 进入面向门级输入的 DFT 插入上下文；没有从功能 RTL 开始做完整逻辑综合 |
| `set_context dft -no_rtl -design_id bscan/edt` | 进行对应阶段的 DFT 插入，并为生成的设计数据指定阶段标识 |
| `set_context dft -scan -design_id scan` | 进入扫描插入上下文，生成标识为 `scan` 的设计版本 |
| `set_context patterns -scan` | 进入扫描测试向量生成上下文 |
| `set_tsdb_output_directory ./tsdb` | 指定 Tessent 数据库目录，用于保存和衔接阶段数据；本流程也从该 TSDB 查找前一阶段设计 |
| `read_cell_library ../lib/adk.tcelllib` | 读取 DFT 单元模型，供分析逻辑、时序单元及扫描等效关系使用 |
| `read_cell_library ../lib/picdram.tcelllib` | 读取存储器相关 DFT 单元模型；具体模型内容需恢复库文件后查看 |
| `set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory` | 告诉工具从 `../lib` 按 `.tcd_memory` 扩展名查找存储器测试描述 |
| `read_verilog 路径` | 直接读入 Verilog 设计；第 1 步读原始网表，第 2 步读插 PAD 的网表 |
| `read_design top -design_id ... -verbose` | 从 TSDB 读取指定设计版本及配套测试信息，输出详细加载过程 |
| `set_current_design $top` | 将 `top` 设为当前分析/编辑的顶层，展开设计并检查模块和连接 |
| `set_system_mode insertion` | 切换到允许进行结构插入/编辑的模式，第 1 步用于加 PAD |
| `set_system_mode analysis` | 建立分析模型并触发相关规则检查，为扫描分析或 ATPG 准备；会执行较多分析，不只是改一个标签 |
| `check_design_rules` | 运行当前上下文的设计规则检查，检查测试结构生成和使用所需的条件 |
| `report_...` | 报告当前状态；通常用于确认设置，本身不等同于生成或修改结构 |

`design_id` 和测试模式名必须分开理解：

| 名称 | 类型 | 在哪里使用 |
|---|---|---|
| `bscan` | 设计阶段标识 | 第 2 步生成，第 3 步读取 |
| `edt` | 设计阶段标识 | 第 3 步生成，第 4 步读取 |
| `scan` | 设计阶段标识 | 第 4 步生成，第 5 步读取 |
| `edt_mode` | 扫描配置模式名；同时关联同名 DFT 使能信号 | 第 4 步 `add_scan_mode`，第 5 步 `import_scan_mode` |
| `edt_stuck` | ATPG 会话/结果模式名 | 第 5 步 `set_current_mode` |

第 3～5 步的 `read_design` 不只是读取一个 `.v`。日志显示它还自动读入 `.vg`、`.icl`、`.pdl`、`.tcd` 等信息，这些数据承载了前一步的测试结构与配置。

## 5. 第一步：1_insert_pad

源文件：[1_insert_pad](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/1_insert_pad:1)。

**输入：** `../netlist/top_netlist.v` 和两个 `.tcelllib`。  
**输出：** `./netlist/top.v`。  
**目标：** 给原有 I/O 加 PAD，并预留 JTAG 和逻辑测试接口。

### 5.1 加载并允许编辑

前 12 行设置顶层、进入门级 DFT 上下文、读库和网表，然后进入 `insertion` 模式。基础命令含义见第 4 节。

### 5.2 为已有输入插入 ipad

```tcl
set inputPortList [get_ports -of_module $top -direction input]
foreach_in_collection inputPort $inputPortList {
    set portName [lindex [get_name_list $inputPort] 0]
    set padName [string map { \[ _ \] _ } ${portName}_PAD]
    create_instance $padName -of_module ipad
    move_connection -from $inputPort -to $padName/C
    create_connection $inputPort $padName/PAD
}
```

| 命令 | 具体作用 |
|---|---|
| `get_ports -of_module $top -direction input` | 取出 `top` 的所有输入端口，按标量位遍历总线 |
| `get_name_list $inputPort` | 将端口对象转换为名称列表 |
| `lindex ... 0` | 取当前端口名称 |
| `string map { \[ _ \] _ } ...` | 将端口名中的方括号改成下划线，例如 `net_in[0]_PAD` 变成 `net_in_0__PAD` |
| `create_instance ... -of_module ipad` | 创建输入 PAD 实例 |
| `move_connection -from ... -to .../C` | 将原来连接到输入端口的内部负载连接迁移到 PAD 的内部侧 C 引脚 |
| `create_connection 端口 PAD/PAD` | 把顶层端口接到 PAD 的外部侧 |

连接关系由“顶层输入直接驱动内部逻辑”变为：

```text
顶层输入端口 → ipad.PAD → ipad.C → 原内部逻辑
```

### 5.3 为已有输出插入 opad

输出部分使用同样的遍历方式，区别是：

```tcl
create_instance $padName -of_module opad
move_connection -from $outputPort -to $padName/I
create_connection $outputPort $padName/PAD
```

形成：

```text
原内部逻辑 → opad.I → opad.PAD → 顶层输出端口
```

这里的 `C`、`I`、`PAD` 来自实验 PAD 模型的引脚定义，不应直接套用到其他工艺的 I/O 单元。

### 5.4 新增测试端口及 PAD

```tcl
set inputPortList {TDI TMS TRST TCK scan_en edt_update test_clock}
set outputPortList {TDO}
```

通过 `foreach`、`create_instance`、`create_port -direction input/output` 和 `create_connection` 为这些端口建立 PAD。

| 端口 | 用途 |
|---|---|
| TDI / TDO | JTAG 串行数据输入/输出 |
| TMS | 控制 TAP 状态转换 |
| TCK | JTAG 测试时钟 |
| TRST | TAP 复位接口；具体有效电平以模型和生成测试过程为准 |
| scan_en | 扫描移位/捕获控制，STIL 的移位宏中设置为 1 |
| edt_update | EDT 动态更新控制 |
| test_clock | 外部逻辑测试时钟，后续派生内部测试时钟 |

新增测试端口在这一步只完成端口到 PAD 外部侧的连接。TAP、EDT 等内部测试逻辑尚未插入，相关内部侧连接留给后续阶段。

### 5.5 保存设计

```tcl
write_design -output_file ./netlist/${top}.v -replace
```

将编辑后的设计写为 `./netlist/top.v`，允许替换同名输出。按日志端口列表计算，本步为原有 41 个 I/O 位及新增 8 个测试端口共创建 **49 个 PAD 实例**。

## 6. 第二步：2_insert_bscan&mbist

源文件：[2_insert_bscan&mbist](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/2_insert_bscan&mbist:1)。

**输入：** 第 1 步的 `./netlist/top.v`。  
**输出：** TSDB 中的 `top_bscan` 设计及测试仪器。  
**目标：** 创建 JTAG/IJTAG 访问网络、边界扫描和存储器自测试硬件。

### 6.1 声明芯片级 DFT 需求

```tcl
set_context dft -no_rtl -design_id bscan
set_tsdb_output_directory ./tsdb
...
set_design_level chip
set_dft_specification_requirements -boundary_scan on -memory_test on
```

`set_design_level chip` 指明按芯片顶层处理；后一句要求工具创建边界扫描与 memory test 相关配置。此时只是声明需求，还没有完成硬件插入。

### 6.2 标记 JTAG 端口功能

```tcl
set_attribute_value TCK -name function -value tck
set_attribute_value TDI -name function -value tdi
set_attribute_value TMS -name function -value tms
set_attribute_value TRST -name function -value trst
set_attribute_value TDO -name function -value tdo
```

这五条把现有端口关联到 JTAG 功能，工具据此连接 TAP 接口。它们不会再创建一组同名顶层端口。

### 6.3 设置边界扫描端口策略

```tcl
set_boundary_scan_port_options clk -cell_options clock
set_boundary_scan_port_options ramclk -cell_options clock
set_boundary_scan_port_options scan_en -cell_options dont_touch
set_boundary_scan_port_options edt_update -cell_options dont_touch
set_boundary_scan_port_options test_clock -cell_options dont_touch
```

- `clock`：按时钟类端口的边界扫描策略处理 clk 和 ramclk。
- `dont_touch`：这些测试控制/时钟端口不按普通功能 I/O 插入边界扫描单元，保留直接测试访问能力。这里是边界扫描选项，不是综合器全局的 `dont_touch` 约束。

### 6.4 声明 scan_en 和功能时钟

```tcl
add_dft_signals scan_en -source_nodes scan_en
add_clocks 0 clk -period 5ns
add_clocks 0 ramclk -period 5ns
```

第一条将 DFT 信号功能 `scan_en` 绑定到现有同名节点。`add_clocks` 的 `0` 是时钟的关闭/非激活状态，`-period 5ns` 声明周期；它不会在会话中直接播放一个波形，也不等同于后面的 STIL 测试周期。

### 6.5 检查规则，创建和修改规格

```tcl
check_design_rules
set spec [create_dft_specification]
report_config_data $spec
```

- `check_design_rules`：先进行设计检查。
- `create_dft_specification`：依据需求、库及设计信息生成结构化 DFT 规格。
- `set spec [...]`：保存返回的配置对象路径；日志为 `/DftSpecification(top,bscan)`。
- `report_config_data`：查看推导出的配置。

历史自动配置识别到：

```text
MemoryBist / Controller(c1)
  时钟域：ramclk
  MemoryInterface(m1)：dram_1
  MemoryInterface(m2)：dram_2
  MemoryInterface(m3)：dram_3
```

脚本没有显式写入算法，但配套 `lib/picdram.tcd_memory` 指定 `Algorithm : SMarchCHKB`，历史 MBIST generation_log 也计算了 SMARCHCHKB 的运行长度。因此本材料可以确认算法来源。没有存储器修复/熔丝配置证据，不能据此断言已实现修复。

随后加入边界扫描辅助 I/O：

```tcl
read_config_data -in ${spec}/BoundaryScan -from_string {
    AuxiliaryInputOutputPorts {
        auxiliary_input_ports : net_in[0], net_in[1] ;
        auxiliary_output_ports : net_out[0], net_out[1] ;
    }
}
report_config_data $spec
```

该设置使选中的边界扫描 I/O 提供辅助访问接口，供第 3 步连接 EDT 通道。它没有新增 `net_in` 或 `net_out` 顶层端口。

### 6.6 实际生成、插入并综合

```tcl
process_dft_specification
extract_icl
run_synthesis
```

| 命令 | 作用 |
|---|---|
| `process_dft_specification` | 校验规格，生成测试仪器 RTL 和描述文件，并把仪器插入当前设计 |
| `extract_icl` | 提取和检查实际 IJTAG 连接关系，输出顶层 ICL 及配套信息 |
| `run_synthesis` | 创建并运行测试仪器的逻辑综合流程，将生成的测试 RTL 映射到单元库并回填 TSDB |

日志明确记录了：

- IJTAG TAP 和 SIB 的生成；
- 边界扫描单元、接口和 `top.bsdl` 的生成；
- MBIST 控制器及存储器接口的生成；
- `top.icl`、`top.pdl`、`top.sdc`、DFT 信息字典的写出；
- 综合结果 `pass 1 fail 0`；
- 合并网表 `./tsdb/dft_inserted_designs/top_bscan.dft_inserted_design/top.vg` 的创建。

## 7. .synopsys_dc.setup 在哪里起作用

文件内容：

```tcl
set link_library /workspace/user/zhaoyixuan/BS_EDT_OOC/internal_netlist/lib/adk.db
set target_library /workspace/user/zhaoyixuan/BS_EDT_OOC/internal_netlist/lib/adk.db
```

| 变量 | 作用 |
|---|---|
| `link_library` | 综合时用于解析和链接设计中引用的单元/模块 |
| `target_library` | 综合映射时可选用的目标工艺单元库 |

`adk.db` 是 Design Compiler 使用的库，`.tcelllib` 是 Tessent 使用的 DFT 模型，两者用途不同，不能仅改一个就认为工具链已配好。

第 2、3 步日志中的综合目录名均含 `synthesis_dc_shell`，说明历史流程调用了 Design Compiler 综合路径。该 `.setup` 是实验提供的 DC 库设置；实际重跑时还应确认生成的 DC 启动脚本在哪个工作目录读取它。原文件使用机器相关的绝对路径，迁移后必须更新。

## 8. 第三步：3_insert_edt_occ

源文件：[3_insert_edt_occ](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/3_insert_edt_occ:1)。

**输入：** TSDB 中 `design_id=bscan` 的设计。  
**输出：** TSDB 中 `design_id=edt` 的设计。  
**目标：** 增加 EDT、OCC 和它们需要的测试控制网络。

### 8.1 读取前一阶段

```tcl
set_context dft -no_rtl -design_id edt
...
read_design top -design_id bscan -verbose
set_current_design $top
```

`edt` 是本步写出的版本，`bscan` 是读入版本。日志确认读取了上一阶段的综合网表 `top.vg` 及测试描述。

### 8.2 声明 DFT 信号

```tcl
add_dft_signals ltest_en
add_dft_signals scan_en edt_update test_clock -source_node {scan_en edt_update test_clock}
add_dft_signals shift_capture_clock edt_clock -create_from_other_signals
add_dft_signals edt_mode
add_dft_signals tck_occ_en
add_dft_signals memory_bypass_en
```

| 信号 | 作用与历史实现 |
|---|---|
| `ltest_en` | 开启逻辑测试；还用来切换四个辅助 I/O 的使能 |
| `scan_en` | 使用既有外部端口控制扫描；第 2 步已添加，日志提示重复添加被忽略 |
| `edt_update` | 使用既有端口提供 EDT 更新控制 |
| `test_clock` | 使用既有端口提供外部测试时钟 |
| `shift_capture_clock` | 从其他已定义测试信号派生，供移位/捕获控制使用 |
| `edt_clock` | 从已定义测试信号派生，供 EDT 使用 |
| `edt_mode` | 选择 EDT 扫描模式；历史实现为 IJTAG TDR 控制位 |
| `tck_occ_en` | 允许 scan-tested 测试仪器在逻辑测试捕获阶段使用 shift_capture_clock；不是简单把外部 TCK 拉高 |
| `memory_bypass_en` | 逻辑测试期间旁路存储器路径，减少 RAM 对逻辑测试的影响 |

`-source_node/-source_nodes` 用于绑定已有源；`-create_from_other_signals` 请求工具根据其他 DFT 信号生成所需信号。没有显式绑定外部节点的静态信号在历史实现中落在 IJTAG TDR 中，不是每声明一个信号都新增一个芯片引脚。第 5 步 `report_dft_signals` 可以看到实现位置。

### 8.3 给辅助 I/O 接入模式切换控制

```tcl
add_dft_control_points [get_auxiliary_pins net_in[0] -enable -direction input] -dft_signal_source_name ltest_en
```

其余三条同理，对 `net_in[1]`、`net_out[0]`、`net_out[1]` 设置：

| 组成 | 含义 |
|---|---|
| `get_auxiliary_pins` | 查找第 2 步形成的边界扫描辅助引脚 |
| `-enable` | 此处获取辅助路径的使能引脚 |
| `-direction input/output` | 区分辅助输入路径和辅助输出路径 |
| `add_dft_control_points ... -dft_signal_source_name ltest_en` | 让这些辅助路径的使能由逻辑测试控制信号驱动 |

日志中的实际目标为 `net_in_0_AuxInEn`、`net_in_1_AuxInEn`、`net_out_0_AuxOutEn`、`net_out_1_AuxOutEn`，均位于边界扫描逻辑组实例内。

### 8.4 创建逻辑测试规格和 IJTAG 分支

```tcl
set_dft_specification_requirements -logic_test On
check_design_rules
report_dft_control_points
set spec [create_dft_specification -sri_sib_list {occ edt}]
report_config_data $spec
```

- `-logic_test On`：声明逻辑测试所需的 DFT 要求。
- `report_dft_control_points`：核对辅助 I/O 和已有测试仪器的控制来源。
- `-sri_sib_list {occ edt}`：为 OCC 和 EDT 建立对应的扫描资源仪器 IJTAG 访问分支，后续用 `Sib(occ)`、`Sib(edt)` 关联仪器。

### 8.5 配置两个 OCC

```tcl
read_config_data -in $spec -from_string {
    OCC {
        ijtag_host_interface : Sib(occ);
    }
}
set id_clk_list [list clk clk ramclk ramclk]
foreach {id clk} $id_clk_list {
    set occ [add_config_element OCC/Controller($id) -in $spec]
    set_config_value clock_intercept_node -in $occ $clk
}
```

| 配置/命令 | 作用 |
|---|---|
| `ijtag_host_interface : Sib(occ)` | 让 OCC 通过对应 IJTAG 分支访问 |
| `foreach {id clk}` | 每次从列表取两个值：控制器 ID、时钟拦截节点 |
| `add_config_element OCC/Controller($id) -in $spec` | 在 DFT 规格中添加 OCC 控制器配置对象 |
| `set_config_value clock_intercept_node ...` | 指定 OCC 要介入控制的时钟位置 |

本例建立 `Controller(clk)` 和 `Controller(ramclk)`。这是两个控制器分别对应两个时钟，不是四个时钟。

### 8.6 配置 EDT 规模

```text
EDT {
    ijtag_host_interface : Sib(edt);
    Controller(c1) {
        longest_chain_range : 2, 100;
        scan_chain_count : 6;
        input_channel_count : 2;
        output_channel_count : 2;
        Connections +{ ... }
    }
}
```

| 参数 | 含义 |
|---|---|
| `Controller(c1)` | 配置一个 EDT 控制器 |
| `longest_chain_range : 2, 100` | EDT 生成时支持的“最长内部扫描链长度”范围，给后续链长度变化留余量；实际最长链为 41 |
| `scan_chain_count : 6` | 内部扫描链数量为 6 |
| `input_channel_count : 2` | 外部压缩输入通道数量为 2 |
| `output_channel_count : 2` | 外部压缩输出通道数量为 2 |
| `Connections +{ ... }` | 补充连接配置，列出需要指定连接位置的输入/输出通道 |

`longest_chain_range` 约束的是最长链的估计范围，不能读成“强制每条链长度为 100”。参数用途可参照 [Siemens Tessent Shell Reference Manual v2021.3，第 2370 页](https://www.lirmm.fr/~virazel/COURS/M2%20-%20HAE926E/Labs/tshell_ref.pdf)。

**6 条内部链和 2 对外部通道是两层概念。** 两者比值为 3，但实验的实际测试数据压缩率还受初始化、掩码、向量数量等影响，不能直接宣布实际压缩率为 3 倍。

### 8.7 将 EDT 通道接到辅助引脚

```tcl
set_config_value port_pin_name \
    -in $spec/EDT/Controller(c1)/Connections/EdtChannelsIn(1) \
    [get_single_name [get_auxiliary_pins net_in[0] -direction input]]
```

`get_auxiliary_pins` 找到辅助数据引脚，此处不带 `-enable`；`get_single_name` 将单个对象转成名称；`set_config_value port_pin_name` 将名称写入连接配置。

四条设置的对应关系如下：

| EDT 通道 | 复用的顶层端口 | 历史辅助引脚末段 |
|---|---|---|
| EdtChannelsIn(1) | net_in[0] | net_in_0_AuxIn |
| EdtChannelsIn(2) | net_in[1] | net_in_1_AuxIn |
| EdtChannelsOut(1) | net_out[0] | net_out_0_AuxOut |
| EdtChannelsOut(2) | net_out[1] | net_out_1_AuxOut |

所以第 2 步的辅助 I/O 声明、第 3 步的使能控制、第 3 步的 EDT 数据连接共同实现了功能引脚复用。

### 8.8 生成、提取和综合

再次执行：

```tcl
process_dft_specification
extract_icl
run_synthesis
```

本步生成额外 IJTAG SIB/TDR、两个 OCC 和一个 EDT。历史综合为 `pass 1 fail 0`，写出：

```text
./tsdb/dft_inserted_designs/top_edt.dft_inserted_design/top.vg
```

**此时 EDT 控制器已存在，但最终的内部扫描链仍要在第 4 步分配和连接。**

## 9. 第四步：4_insert_scan

源文件：[4_insert_scan](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/4_insert_scan:1)。

**输入：** TSDB 中 `design_id=edt` 的设计。  
**输出：** TSDB 中 `design_id=scan` 的设计。  
**目标：** 创建扫描配置、分配内部链并实际插入连接。

### 9.1 读取并分析

```tcl
set_context dft -scan -design_id scan
...
read_design $top -design_id edt -verbose
set_current_design $top
set_system_mode analysis
report_scan_segments
report_clocks
```

- `report_scan_segments`：查看已存在、可复用的扫描段。历史记录包括一个 7 单元段及两个 OCC 各 3 单元的段；它们不是最终 6 条完整扫描链。
- `report_clocks`：确认功能时钟、测试时钟以及从 OCC 推导出的捕获时钟。扫描阶段列出 8 个时钟项，并不意味着芯片新增了 8 个外部时钟端口。

### 9.2 自动找到 EDT 实例

```tcl
set edt_instance [get_name_list [get_instance -of_module [get_name [get_icl_module -filter tessent_instrument_type==mentor::edt]]]]
```

由内向外理解：

1. `get_icl_module -filter tessent_instrument_type==mentor::edt`：在 ICL 中筛选 EDT 仪器模块。
2. `get_name`：取得模块名称。
3. `get_instance -of_module ...`：找出该模块在设计中的实例。
4. `get_name_list`：将实例对象转换为名称列表。
5. `set edt_instance ...`：保存到变量。

历史返回 `top_edt_tessent_edt_c1_inst`。这样写可以避免直接硬编码工具生成的实例名。

### 9.3 声明扫描模式，分配并报告链

```tcl
add_scan_mode edt_mode -edt_instance $edt_instance
analyze_scan_chains
report_scan_chains
```

| 命令 | 作用 |
|---|---|
| `add_scan_mode` | 创建名为 edt_mode 的扫描配置，并关联 EDT 实例；日志确认从 EDT 描述中读取链数 6，并关联同名使能信号 |
| `analyze_scan_chains` | 根据可扫描单元、已有扫描段和模式约束，完成链分配方案 |
| `report_scan_chains` | 查看每条链的输入、输出和长度 |

历史结果：

| 链名 | 长度 |
|---|---:|
| chain0 | 41 |
| chain1 | 41 |
| chain2 | 41 |
| chain3 | 41 |
| chain4 | 40 |
| chain5 | 40 |
| 合计 | 244 |

每条链接在 EDT 的 `edt_scan_in[i]` 和 `edt_scan_out[i]` 之间。244 是最终链中识别到的扫描单元数量，包含既有扫描段等内容，不能视为“原始功能设计中有 244 个寄存器”。

### 9.4 实际插入

```tcl
insert_test_logic
```

该命令实现扫描转换、链连接和所需辅助逻辑。`analyze_scan_chains` 负责规划，这一步才落实结构。

历史插入汇总：

- 新增顶层端口：0；
- 新增实例：17；
- 新增 retiming 逻辑：9；
- 新增扫描链：6；
- 网表及相关文件保存到 `./tsdb/dft_inserted_designs/top_scan.dft_inserted_design/`。

第 5 步日志确认从该目录读入 `top.vg`。本脚本没有单独写 `write_design`，是因为 `insert_test_logic` 在这套 TSDB 流程中已写出对应数据。

## 10. 第五步：5_atpg

源文件：[5_atpg](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/5_atpg:1)。

**输入：** TSDB 中 `design_id=scan` 的设计。  
**目标：** 在 EDT 模式下生成 stuck-at 测试向量，统计并导出。

### 10.1 加载扫描设计和模式

```tcl
set_context patterns -scan
...
read_design $top -design_id scan -verbose
set_current_design $top
set_current_mode edt_stuck
report_dft_signals
import_scan_mode edt_mode
report_core_descriptions
report_clocks
report_dft_signals
```

| 命令 | 作用 |
|---|---|
| `set_current_mode edt_stuck` | 为当前 ATPG 会话及结果选择模式名 |
| `report_dft_signals` | 核对 DFT 信号是否存在、来源类型及实现节点 |
| `import_scan_mode edt_mode` | 导入第 4 步建立的扫描配置、EDT/OCC 描述和相关设置 |
| `report_core_descriptions` | 查看当前设计及测试 IP 的测试描述，例如 EDT 和两个 OCC |
| `report_clocks` | 查看当前 ATPG 模式中的时钟；历史报告的用户定义时钟为 test_clock |

`edt_stuck` 的命名便于识别用途，但不能把自定义名字当作任意故障模型切换命令。此处使用默认 stuck-at 模型，且历史输出明确确认该模型。要切换故障类型应使用相应的 `set_fault_type` 等设置，详见 [Siemens Reference Manual v2021.3 的 set_fault_type](https://www.lirmm.fr/~virazel/COURS/M2%20-%20HAE926E/Labs/tshell_ref.pdf)。

### 10.2 设置并核对静态控制值

```tcl
set_static_dft_signal_values tck_occ_en 1
report_static_dft_signal_settings
```

这让 scan-tested 仪器在逻辑测试捕获阶段可以使用相应测试时钟。历史静态设置为：

| DFT 信号 | 值 | 日志中的来源 |
|---|---:|---|
| tck_occ_en | 1 | 显式设置 |
| ltest_en | 1 | 工具推导 |
| memory_bypass_en | 1 | 工具推导 |
| edt_mode | 1 | 扫描模式设置 |

`pattern.stil` 的 JTAG 初始化段也列出了这四个控制位，说明这些设置最终被转换为可从 TAP 访问的配置操作。

### 10.3 进入分析并生成向量

```tcl
set_system_mode analysis
create_patterns
```

进入分析模式后，工具进行扫描链追踪、测试初始化过程仿真、EDT 检查、时钟/RAM 规则检查等工作。日志确认一个 EDT 块、6 条链、244 个扫描单元，最长链 41。

`create_patterns` 根据当前模式、约束和故障模型执行 ATPG。历史日志提示故障表为空并自动加入全部故障，因此本脚本没有显式 `add_faults -all` 仍然生成了故障表。

工具还根据分析自动调整了 split capture、abort limit 和 sequential depth。日志出现 `set_pattern_type -sequential 2`，这解释了为什么结果同时包含 basic 和 clock_sequential 向量。**clock_sequential 是向量类型，不能据此把本次结果改称 transition 测试。**

### 10.4 报告覆盖率和扫描数据量

```tcl
report_statistics -detailed_analysis
report_scan_volume
```

- `report_statistics -detailed_analysis`：打印故障分类、未测原因、覆盖率及向量数量，帮助定位覆盖率损失。
- `report_scan_volume`：打印外部通道数、移位周期、各类型向量数量及数据量。

结果详见第 11 节。

### 10.5 当前脚本的导出命令

```tcl
write_patterns patterns/pattern.stil -stil -replace
write_patterns patterns/top_stuck_parallel.v -verilog -parallel -replace -scan -parameter_list {SIM_KEEP_PATH 1}
set_pattern_filtering -sample_per_type 2
write_patterns patterns/top_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}

# write_tsdb_data -replace
```

| 命令/选项 | 作用 |
|---|---|
| `write_patterns 路径` | 将当前向量集按指定格式写出 |
| `-stil` | 输出 STIL 测试交换格式 |
| `-verilog` | 输出用于仿真的 Verilog 测试平台/向量文件，实际可能还有配套数据文件 |
| `-parallel` | 以并行装载扫描单元的形式输出，便于较快验证捕获和响应 |
| `-serial` | 按串行移位过程输出，便于验证实际扫描移位路径和控制时序 |
| `-scan` | 原脚本选择 scan/ATPG 测试向量的旧式写法；2021.3 参考手册的正式写法是 `-pattern_sets scan`。当前日志没有验证这条导出命令，重跑时应核对本机版本支持情况 |
| `-replace` | 允许覆盖相应输出 |
| `-parameter_list {SIM_KEEP_PATH 1}` | 让生成的 Verilog testbench 引用 pattern 数据文件时保留导出命令中给出的完整路径；默认 0 会去掉路径信息。它影响数据文件查找，不会保留或修改 DUT 的逻辑层次 |
| `set_pattern_filtering -sample_per_type 2` | 对后续导出的 scan/ATPG 向量按类型抽样，每类取 2 个；不重新执行 ATPG |
| `write_tsdb_data -replace` | 将当前 ATPG 模式的测试描述、分析模型、故障和向量数据库等写入 TSDB；当前脚本此行被注释，不执行 |

过滤发生在 STIL 和并行 Verilog 导出之后，因此当前脚本预期先导出完整 STIL 和 scan 并行向量，再导出抽样串行向量。过滤只作用于 scan/ATPG 向量，不作用于 chain test；“每类 2 个”不代表整个串行文件只有 2 个向量。需要恢复完整导出时可使用 `set_pattern_filtering -off`。这些输出与过滤语义可核对 [Siemens Reference Manual v2021.3 的 write_patterns 和 set_pattern_filtering](https://www.lirmm.fr/~virazel/COURS/M2%20-%20HAE926E/Labs/tshell_ref.pdf)。

例如命令输出到 `patterns/top_stuck_parallel.v`，保留路径设置会让测试平台按命令给出的路径寻找配套数据文件；如果导出时给出的是相对路径，它仍依赖仿真工作目录，不能理解成自动转换为绝对路径。`SIM_KEEP_PATH` 的定义见上述手册第 4698～4699 页。并行 EDT Verilog 文件针对内部扫描链装载；STIL 中的扫描数据则对应外部通道，两种表示服务于不同用途。

**生成 Verilog 测试平台不等于已经跑过仿真。** E 盘 `flow/patterns/` 保存了对应串行/并行文件及 `.vec/.cfg`，但 5_atpg.log 没有这些导出命令的记录，也未发现对应 ATPG testbench 的仿真通过日志。

## 11. 历史结果应该怎样解读

主要依据：[5_atpg.log](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/5_atpg.log:319)。

### 11.1 结构与向量结果

| 项目 | 历史结果 |
|---|---:|
| EDT 控制器 | 1 |
| OCC 控制器（clk、ramclk） | 2 |
| 外部输入/输出通道 | 2 / 2 |
| 内部扫描链 | 6 |
| 扫描单元总数 | 244 |
| 最长链长度 | 41 |
| basic 向量 | 132 |
| clock_sequential 向量 | 32 |
| ATPG 测试向量 | 164 |
| 另加 chain_test 向量 | 6 |
| 扫描数据量报告中的总向量数 | 170 |
| ATPG 过程中仿真过的向量数 | 193 |

`164 = 132 + 32`；`170 = 164 + 6`。193 是生成过程中的仿真数量，不能当作最终保留的测试向量数。

### 11.2 两种覆盖率口径

| 指标 | total：全部故障 | total relevant：相关故障 |
|---|---:|---:|
| 故障总数 | 18,964 | 16,401 |
| test_coverage | 83.27% | 97.07% |
| fault_coverage | 79.17% | 91.54% |
| atpg_effectiveness | 99.97% | 99.97% |

两列对故障统计范围不同；同时，test_coverage 与 fault_coverage 对不可测故障的计入方式也不同。比较报告时必须同时给出指标名和列名。

本日志中，从 relevant 口径剔除的 AU 子类为：

```text
EDT    65
OCC  1060
IJTAG 1438
合计 2563

18964 - 2563 = 16401
```

因此“83.27% 提升到了 97.07%”并不是两次优化结果，而是同一次运行的两种统计口径。STIL 头部记录的是 total 列的 **83.27% TC / 79.17% FC**。

`atpg_effectiveness=99.97%` 也不能解释为“99.97% 的全部故障已经被检测”。工具对已分类、已判定不可测等故障的处理会影响该指标。关于跨模式和 DFT 仪器的覆盖率分类，可参照 [Siemens 的覆盖率统计说明](https://blogs.sw.siemens.com/tessent/2019/07/02/how-to-create-comprehensive-test-coverage-reports-during-hierarchical-dft/)。

### 11.3 主要故障类别

| 类别 | 日志数量（total） | 含义 |
|---|---:|---|
| DS / det_simulation | 9,930 | 由仿真确认检测 |
| DI / det_implication | 5,081 | 由工具的蕴含/结构规则等判定为检测，不全是当前向量逐一仿真检测 |
| UO / unobserved | 6 | 未观察到故障效应 |
| PU / posdet_untestable | 5 | 可能检测相关分类；不能直接全部算作确定检测 |
| UU / unused | 758 | 无用逻辑相关故障 |
| TI / tied | 37 | 常量连接相关故障 |
| BL / blocked | 27 | 故障效应被阻断 |
| RE / redundant | 112 | 被判为冗余 |
| AU / atpg_untestable | 3,008 | 当前 ATPG 模式、约束或能力下不可测；relevant 列为 445 |

不要把每个 AU 都当作物理冗余，也不要把 DI 中的 `MBIST` 数字理解成已完成 RAM 阵列自测试的证据。

### 11.4 移位长度与数据量

历史 `report_scan_volume`：

```text
channels     : 2
shift cycles : 41+9
chain_test   : 6     →   600
basic        : 132   → 13200
clock_seq    : 32    →  3200
total        : 170   → 17000
```

41 对应最长内部扫描链；额外 9 个周期是该 EDT 流程的附加移位开销。日志还提示 EDT 初始化使每个向量大小增加约 22%，与 `9/41` 对应。

按日志的统计方式，`2 × (41 + 9) × 170 = 17,000`。这是报告中的扫描 volume 单位（cell loads or unloads），不是完整 STIL 文件大小，也不是总测试时间；JTAG 初始化、捕获周期及末尾卸载等还需要另行考虑。

## 12. pattern.stil 怎样阅读

源文件：[pattern.stil](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/pattern.stil:1)。

| 区段 | 本文件内容和作用 |
|---|---|
| `STIL 1.0` | 文件格式版本 |
| `Header` | 标题、2022-06-27 生成时间、Tessent 2021.3 来源、STUCK 模型和覆盖率 |
| `Signals` | 定义芯片输入/输出信号及方向 |
| `SignalGroups` | 定义 `_pi_`、`_po_`、时序组、两个 EDT 输入/输出通道组 |
| `Timing STUCK_timing` | 给出波形表 `tset_gen_tp1`，周期为 40ns |
| `ScanStructures` | 表示两个 EDT 外部通道，每个 ScanLength 为 50，测试主时钟为 test_clock |
| `MacroDefs` | 定义 `scan_edt_grp1` 移位宏，宏内设置 scan_en、edt_update 和通道数据 |
| `PatternBurst` / `PatternExec` | 指定要执行的 pattern 和所用时序 |
| `Pattern scan_test` | 具体初始化、链测试、逻辑测试及最后卸载操作 |

几个容易混淆的地方：

1. **STIL 有两个 ScanChain 描述，不代表内部只有两条链。** 这里描述的是两个 EDT 外部通道，内部 6 条链由 EDT 展开和压缩响应。
2. **ScanLength=50 不代表每条内部链都长 50。** 它与日志的 `41+9` 外部移位长度一致。
3. **40ns 与前面 5ns 不同。** 5ns 是前期功能时钟声明，40ns 是导出向量实际采用的测试时序表；不能用 5ns 推断本文件测试周期。
4. **Header 的 pattern_end=163 是 ATPG 逻辑测试编号 0～163。** 文件还含单独编号的 6 个 chain pattern，不能只看这个字段统计全部测试类型。

具体执行内容：

```text
TAP 复位和进入 Idle
  → 通过 Shift-IR/Shift-DR 访问 IJTAG 分支及控制寄存器
  → 设置 edt_mode/ltest_en/memory_bypass_en/tck_occ_en 等
  → 运行链测试
  → 运行 pattern 0～131 的 basic 测试
  → 运行 pattern 132～163 的 clock_sequential 测试
  → 卸载最后一次捕获响应
```

数据符号结合当前波形表阅读：

- 输入的 `0/1` 表示驱动值；
- 输出 `L/H` 表示期望比较低/高；
- `X` 表示该处不进行有效值比较，不能直接当成确定逻辑 0；
- `\r50 X` 之类是重复压缩记法，表示重复相应数量的符号；
- 宏定义中的 `#` 是调用时替换的数据位置，与 Tcl 脚本注释的 `#` 用法不同；
- `Macro "scan_edt_grp1"` 把本次输入比特流和预期响应传入移位宏。

文件中的移位通常同时装入下一次测试状态并卸载前一次响应，所以某次宏调用的输出并不一定是“这次刚输入的数据”的响应。

## 13. 脚本与日志的差异

重点对照：[5_atpg](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/5_atpg:33) 和 [5_atpg.log](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/5_atpg.log:400)。

| 项目 | 当前脚本 | 历史日志/当前保存文件 |
|---|---|---|
| STIL 输出位置 | `patterns/pattern.stil` | 根目录和 patterns 下两份均存在，主日志记录写根目录 |
| 并行 Verilog 导出 | 有命令 | 主日志没有执行记录，但 E 盘 patterns 下有实际输出 |
| 按类型抽样 | 有命令 | 日志没有执行记录 |
| 串行 Verilog 导出 | 有命令 | 主日志没有执行记录，但 E 盘 patterns 下有实际输出 |
| ATPG 写回 TSDB | `write_tsdb_data` 被注释 | 历史日志实际执行，并列出 `.tcd.gz`、`.flat.gz`、`.patdb`、`.faults.gz` 输出 |

说明脚本和日志不是完全一致的最终版本。可以据此确认历史做过 STIL 导出和 ATPG 数据保存，但不能宣称当前脚本已原样验证通过，也不能宣称 Verilog 仿真已经通过。

## 14. 日志中的警告和验证边界

### 14.1 边界扫描引脚顺序

第 2 步没有读入 DEF，工具按端口声明顺序创建 `top.pinorder`。这能够支持实验流程，但不能代表边界扫描链已经按实际 PAD 物理位置排序。

### 14.2 浮空、未驱动及 FN/FP 规则

日志出现 Floating input、Undriven net、Net not connected，以及 FN1/FN4/FP13 警告。它们贯穿若干阶段，需要结合完整网表和版本手册定位。当前日志没有足够细节，不能只凭规则编号一概判为无害，也不能直接认定后续结果无效。

工具提示可用 `set_current_design` 的 `-show_elaboration_warnings` 选项查看展开警告详情。E 盘原网表和 TSDB 均存在，可在恢复匹配工具环境后进一步定位；本次没有重新展开设计。

### 14.3 MBIST 生成警告

第 2 步明确要求检查：

```text
./tsdb/instruments/top_bscan_mbist.instrument/top_bscan_tessent_mbist_c1.generation_log
```

该文件实际存在，记录 1 条 PPHW01 警告：未指定 NumberOfDataRegisterBits，针对当前 HardProgrammable、SMarch/SMarchCHKB 配置，工具将 expect/write 数据寄存器设置为最小 2 位。应检查这是预期配置，而不是把它混同于 RAM 的 8 位数据宽度。

### 14.4 扫描等效单元与 non-scan 逻辑

第 4 步提示 `nlatch` 缺少 muxscan 等效模型，并列出若干被视为 non-scan 的单元。这影响哪些时序单元可以加入扫描链，不等于全部寄存器都完成扫描转换。

### 14.5 时钟与锁存器规则

存在 D6、D7、C3、C4、C7 等记录，例如锁存器不透明、时钟稳定性或 non-scan 捕获能力检查。ATPG 自动增加了一些处理，但这不是对全部时钟结构的独立签核。

### 14.6 RAM 相关警告

扫描/ATPG 阶段出现 A6，以及 3 个 RAM 不可观察、不能写入的 A14/A16 记录。历史静态配置中 `memory_bypass_en=1`，因此这些现象与当前逻辑 ATPG 绕过存储器的模式一致；**仍需独立的 MBIST 测试与验证来证明存储器阵列可测**。E 盘有 MemoryBist_P1 和 ParallelRetentionTest 的生成测试平台/PDL，但没有找到对应执行 transcript，不能确认阵列测试仿真已通过。

### 14.7 并行测试平台的库子链警告

ATPG 日志提示有 3 个 library sub-chain，并行 testbench 每次 load 需要额外移位。日志提出 `set_parallel_load_subchain on` 作为可能设置，同时要求 ATPG 库与仿真库引脚一致。不能只为消除警告就盲目打开该选项。

历史流程能够证明：测试结构生成、两次仪器综合通过、扫描插入完成、ATPG 生成和 STIL 写出。它没有证明：门级串/并行仿真通过、MBIST 阵列测试通过、物理时序签核通过或 transition 测试通过。

## 15. 迁移工具环境后的操作顺序

以下是供复现使用的操作说明，本次没有执行。命令按历史 **Linux/Bash + Tessent** 环境书写，不是 Windows PowerShell 的完整可运行方案。

### 15.1 准备工作

保留现有输入和 TSDB，在可写实验副本中核对/迁移：

- Tessent 及其许可证，优先匹配 2021.3；
- Synopsys Design Compiler 及其许可证、可执行路径；
- `../lib/adk.tcelllib`、`../lib/picdram.tcelllib`；
- 存储器 `.tcd_memory` 描述及其依赖；
- `../netlist/top_netlist.v`；
- `.synopsys_dc.setup` 中实际存在、与 DFT 模型匹配的 `.db` 库路径。

在 Linux 版本的实验目录中准备脚本所需的显式输出目录：

```bash
cd /实际材料路径/flat_flow_netlist/flow
mkdir -p netlist patterns
```

所有脚本的 `../lib`、`./tsdb` 等相对路径都依赖工具启动时的工作目录。

### 15.2 第一步

Linux Shell：

```bash
tessent -shell -log 1_insert_pad.log -replace
```

进入 Tessent 后：

```tcl
dofile 1_insert_pad
exit
```

确认 `./netlist/top.v` 生成，再进入下一步。

### 15.3 第二步

Linux Shell：

```bash
tessent -shell -log '2_insert_bscan&mbist.log' -replace
```

Tessent：

```tcl
dofile {2_insert_bscan&mbist}
exit
```

在 Bash 中，未加引号的 `&` 会被解释为后台执行符号。Tessent 中用大括号明确将它作为一个文件名参数。检查综合通过以及 `top_bscan.../top.vg` 和配套 TSDB 数据是否完整。

### 15.4 第三步

Linux Shell：

```bash
tessent -shell -log 3_insert_edt_occ.log -replace
```

Tessent：

```tcl
dofile 3_insert_edt_occ
exit
```

核对 EDT 参数、四个辅助连接、两个 OCC，以及综合成功和 `top_edt.../top.vg`。

### 15.5 第四步

Linux Shell：

```bash
tessent -shell -log 4_insert_scan.log -replace
```

Tessent：

```tcl
dofile 4_insert_scan
exit
```

核对链分配、扫描插入汇总和 `top_scan` 阶段数据。若网表或库不同，不能要求链长一定与历史相同。

### 15.6 第五步

Linux Shell：

```bash
tessent -shell -log 5_atpg.log -replace
```

Tessent：

```tcl
dofile 5_atpg
exit
```

当前脚本将写到 `patterns/`。如需保存可恢复的 ATPG 数据，应在结束会话前执行脚本中已给出的 `write_tsdb_data -replace`，或在后续修改脚本时取消该行注释。本次保留原实验脚本，没有更改它的执行行为。

生成后按同一次运行核对日志、STIL 头部、故障模型、向量数量和两列覆盖率。随后需要用完整扫描网表和匹配的 Verilog 仿真库编译/仿真测试平台；本材料提供 `lib/adk.v` 和 `lib/picdram.v` 等模型，可作为编译清单的输入；具体命令仍取决于仿真器版本、设计依赖和生成 testbench，现有材料没有一条已验证的 ATPG 门级仿真运行命令。

## 16. 读实验时最值得追踪的四条关系

1. **设计版本：** 原始网表 → PAD 网表 → bscan → edt → scan → ATPG 模式 edt_stuck。
2. **数据通路：** net_in[0:1] 的辅助入口 → EDT 解压 → 6 条内部链 → EDT 响应压缩 → net_out[0:1] 的辅助出口。
3. **控制通路：** JTAG TAP → IJTAG SIB/TDR → edt_mode、ltest_en、memory_bypass_en、tck_occ_en 等控制位。
4. **时钟通路：** 功能 clk/ramclk 和外部 test_clock → 相应测试时钟生成/选择与 OCC → 扫描移位、捕获及测试仪器控制。

这四条关系分别由不同命令建立，不能只看 ATPG 的最后一个 `create_patterns` 就理解整个实验。


## 17. 完整材料补充：功能电路、库和现存输出

### 17.1 功能数据通路

来源：[top_netlist.v](E:/tessent/Reference_2026/tessent/flat_flow_netlist/netlist/top_netlist.v)。

```text
net_in[7:0] → dram_1 ─┐
net_in[15:8] → dram_2 ├→ add_1 → add_2 → dram_3 → 8 个 dff → net_out
dram_2 输出 → 8 个 dff ────────────↑
```

两个 adder 的 cin/cout 都是 8 位向量，由异或、与或逻辑按位实现，不能未经核对就把它当作普通单一进位输入的 ripple-carry 加法器。

### 17.2 三种库各自负责什么

| 类型 | 实际文件 | 用途 |
|---|---|---|
| DFT 逻辑库 | adk.tcelllib、picdram.tcelllib | Tessent 分析与扫描映射 |
| 存储器测试描述 | picdram.tcd_memory | 70×8、SRAM 1RW、算法/端口/地址映射 |
| 综合库 | adk.db | DC 单元链接和映射 |
| 仿真库 | adk.v、picdram.v | Verilog 正常功能模型 |

尤其不要因为文件名为 picdram 就把 MemoryType 自动解释为 DRAM；本包的测试描述明确写的是 SRAM。

### 17.3 输出已经存在，但来源证据要分开

`flow/patterns/pattern.stil` 与根目录 pattern.stil 均保存；`top_stuck_parallel.v`、`top_stuck_serial.v` 及各自 vec/cfg/名称文件也存在。这比此前只有 12 个文件的 f_f_n 副本更完整。

主流程 5_atpg.log 仍然只记录根目录 STIL 写出和 TSDB 保存，未记录 Verilog 导出。因此文件内容可以作为独立产物分析，不能修改历史日志所证明的执行顺序。

独立核对入口：

- [存储器测试描述](E:/tessent/Reference_2026/tessent/flat_flow_netlist/lib/picdram.tcd_memory)；
- [存储器仿真模型](E:/tessent/Reference_2026/tessent/flat_flow_netlist/lib/picdram.v)；
- [MBIST 生成日志](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/tsdb/instruments/top_bscan_mbist.instrument/top_bscan_tessent_mbist_c1.generation_log)；
- [并行 ATPG 测试平台](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/patterns/top_stuck_parallel.v)；
- [串行 ATPG 测试平台](E:/tessent/Reference_2026/tessent/flat_flow_netlist/flow/patterns/top_stuck_serial.v)。
