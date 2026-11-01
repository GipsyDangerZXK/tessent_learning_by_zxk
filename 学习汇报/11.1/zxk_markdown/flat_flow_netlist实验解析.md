# flat_flow_netlist 实验解析

## 1. Flat Design 的四大环节

四大环节：

1. Performing DFT hardware insertion：插入可测试性硬件
2. Synthesis：综合新增的 DFT 逻辑
3. Scan insertion (optional)：插入扫描链
4. Gate-level ATPG：在门级设计上生成测试向量

## 2. 实验概述

从门级网表开始，采用平坦式 DFT 流程。  
flat：以顶层设计为整体开展 DFT 和 ATPG，不是先做各个核心的独立 ATPG 再在顶层重定向。  
flow：一套有先后依赖关系的处理流程。  
netlist：输入是已经综合的门级网表，而不是行为级 RTL。  
注意：flat 描述的是测试组织方式，不等于设计文件中绝对不能有模块层次。

### 五个主步骤

- **第一步：`1_insert_pad`** — 插入输入/输出 PAD，预留 JTAG 和扫描测试控制端口。

- **第二步：`2_insert_bscan&mbist`** — 加入 TAP、Boundary Scan（边界扫描）、IJTAG 访问结构和 MBIST（存储器内建自测试）。

- **第三步：`3_insert_edt_occ`** — 加入扫描压缩 EDT（Embedded Deterministic Test）、片上时钟控制器 OCC（On-Chip Clock Controller）。

- **第四步：`4_insert_scan`** — 分析、分配并连接内部扫描链。

- **第五步：`5_atpg`** — 读取扫描设计，生成逻辑故障测试向量并导出。

### 文件交接关系

```text
../netlist/top_netlist.v
    ↓ 第一步插 PAD
./netlist/top.v
    ↓ 第二步插 Boundary Scan、MBIST，并综合新增仪器
./tsdb 中的 top / design_id=bscan
    ↓ 第三步插 EDT、OCC，并综合新增仪器
./tsdb 中的 top / design_id=edt
    ↓ 第四步插扫描链
./tsdb 中的 top / design_id=scan
    ↓ 第五步 ATPG
patterns/pattern.stil
patterns/top_stuck_parallel.v 及配套数据文件
patterns/top_stuck_serial.v 及配套数据文件
```

## 3. 模块化代码阅读理解

下面将“一”视为第一步，“二”视为第二步，以此类推。

### 模块一：初始化——建立变量、选择上下文和数据库目录

首先，建立变量 top，其值为字符串 top

```tcl
set top top
```

set 是 Tcl 的赋值命令；后文 `$top` 表示取出变量的值，即 top  
`${top}.v` 表示将变量 top 的值与后面的 .v 拼接，得到 `top.v`

随后，进入对应阶段的工作上下文

| 阶段 | 指令 |
| --- | --- |
| 一 | `set_context dft -no_rtl` |
| 二 | `set_context dft -no_rtl -design_id bscan` |
| 三 | `set_context dft -no_rtl -design_id edt` |
| 四 | `set_context dft -scan -design_id scan` |
| 五 | `set_context patterns -scan` |

dft：进入 DFT 设计处理上下文  
`-no_rtl`：当前以非 RTL、即本实验的门级设计为处理起点  
dft `-scan`：进入扫描插入相关上下文  
patterns `-scan`：进入扫描测试向量生成上下文  
`-design_id`：标记本次阶段的设计版本，区分 bscan、edt、scan  
`design_id` 不会把顶层模块 top 改名为 bscan、edt 或 scan

指定 TSDB 数据库的输出位置；后续 `read_design` 也依赖相应的 TSDB 设计数据

| 阶段 | 指令 |
| --- | --- |
| 二、三、四、五 | `set_tsdb_output_directory ./tsdb` |

TSDB 是 Tessent 保存设计版本、测试仪器及测试元数据的数据库目录  
不是所有产物都自动放入 TSDB：第一步网表、第五步显式导出的 patterns 文件有各自路径  
./ 与 ../ 都相对于启动工具时的工作目录；本实验应以 flow 作为工作目录

### 模块二：读取库、读取输入设计、指定当前顶层

读取 Tessent 单元库和存储器单元库

```tcl
read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
```

`adk.tcelllib` 提供本实验工艺单元的测试模型  
`picdram.tcelllib` 提供相应存储器单元的测试模型  
这些库用于 Tessent 对设计进行识别和分析，不等于 DC 综合使用的 .db 库

指定存储器测试描述文件的查找规则

```tcl
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
```

`-format` `tcd_memory`：查找的源类型是存储器测试描述  
`-Y` `../lib`：在 `../lib` 目录中查找  
`-extension` `tcd_memory`：对应文件扩展名为 `tcd_memory`  
这条命令建立源文件查找方式，不是读取功能 RTL 的 `read_verilog` 命令

读取当前步骤实际需要的输入设计

| 阶段 | 指令 |
| --- | --- |
| 一 | `read_verilog ../netlist/top_netlist.v` |
| 二 | `read_verilog ./netlist/${top}.v` |
| 三 | `read_design top -design_id bscan -verbose` |
| 四 | `read_design $top -design_id edt -verbose` |
| 五 | `read_design $top -design_id scan -verbose` |

第一步读取最初的门级网表  
第二步读取第一步输出的带 PAD 网表  
第三、四、五步读取前一阶段在 TSDB 中保存的设计与相关测试数据  
`-verbose`：显示更详细的读取信息  
`read_design` 交接的不只是一个 Verilog 文件，还包括工具所需的测试元数据

把当前要操作的设计设置为顶层 top

```tcl
set_current_design $top
```

第三步原文直接写 `read_design` top，与本例 `$top` 的值相同  
第二步的 `./netlist/top.v` 与第一步输入的 `../netlist/top_netlist.v` 是不同文件

### 模块三：第一步——进入编辑模式，为原有输入端口插入 PAD

允许进行实例、端口及连接的插入和修改

```tcl
set_system_mode insertion
```

收集 top 模块的全部输入端口

```tcl
set inputPortList [ get_ports -of_module $top -direction input ]
```

遍历 Tessent 返回的输入端口集合，并逐个插入 ipad

```tcl
foreach_in_collection inputPort $inputPortList {
    set portName [lindex [get_name_list $inputPort] 0]
    set padName  [ string map { \[ _ \] _ } ${portName}_PAD ]
    create_instance $padName -of_module ipad
    move_connection -from $inputPort -to $padName/C
    create_connection $inputPort $padName/PAD
}
```

`foreach_in_collection` 用于遍历工具的对象集合  
`get_name_list` 将端口对象转换为名称列表  
lindex ... 0 取该列表的第一个名称，保存到 portName  
string map 将端口名中的 [、] 替换为下划线，以便构造 PAD 实例名  
例如：`net_in[0]`_PAD → `net_in_0__PAD`，末尾可能出现两个连续下划线  
`create_instance` 创建一个 ipad 单元实例，不是新增一个同名顶层端口  
`move_connection` 把原来接在输入端口上的内部负载连接，移到 ipad 的 C 引脚  
`create_connection` 再把顶层输入端口接到 ipad 的 PAD 引脚

插入前：顶层输入端口 → 内部功能逻辑  
插入后：顶层输入端口 → `ipad.PAD` → `ipad.C` → 内部功能逻辑  
ipad 是本实验采用的输入 PAD 模型；引脚名称要以实际库模型为准

### 模块四：第一步——为原有输出端口插入 PAD

收集 top 模块的全部输出端口

```tcl
set outputPortList [ get_ports -of_module $top -direction output ]
```

逐个插入 opad，并调整原有输出连接

```tcl
foreach_in_collection outputPort $outputPortList {
    set portName [lindex [get_name_list $outputPort] 0]
    set padName  [ string map { \[ _ \] _ } ${portName}_PAD ]
    create_instance $padName -of_module opad
    move_connection -from $outputPort -to $padName/I
    create_connection $outputPort $padName/PAD
}
```

与输入 PAD 的操作相似，但方向相反  
原来驱动顶层输出端口的内部逻辑，现在改为驱动 opad 的 I 引脚  
opad 的 PAD 引脚再连接顶层输出端口

插入前：内部功能逻辑 → 顶层输出端口  
插入后：内部功能逻辑 → `opad.I` → `opad.PAD` → 顶层输出端口  
本脚本分别处理 input 与 output，没有单独编写 inout 双向端口的插入循环

### 模块五：第一步——预留测试端口并输出带 PAD 的网表

重新赋值 inputPortList 和 outputPortList，此时存的是 Tcl 名称列表  
它们与前面 `get_ports` 返回的工具对象集合不同，因此后面使用普通 foreach

```tcl
set inputPortList { TDI TMS TRST TCK scan_en edt_update test_clock}
set outputPortList { TDO }
```

为每个新增测试输入创建 ipad、顶层端口，以及端口到 PAD 的连接

```tcl
foreach portName $inputPortList {
    set padName ${portName}_PAD
    create_instance $padName -of_module ipad
    create_port $portName -direction input
    create_connection $portName $padName/PAD
}
```

为新增测试输出 TDO 创建 opad 和顶层端口

```tcl
foreach portName $outputPortList {
    set padName ${portName}_PAD
    create_instance $padName -of_module opad
    create_port $portName -direction output
    create_connection $portName $padName/PAD
}
```

这时主要是预留接口；脚本没有在这里手工连接新增 PAD 的全部内部侧引脚  
后续 DFT 插入根据端口属性、DFT 信号声明和库模型完成相应集成  
因此“有 TDI/TDO 端口”与“已经插入 TAP 控制器”是两个阶段的事情

各测试端口的用途  
TDI：JTAG 串行测试数据输入  
TDO：JTAG 串行测试数据输出  
TMS：JTAG TAP 状态机的模式选择输入  
TCK：JTAG 测试时钟  
TRST：JTAG TAP 复位端口；有效极性应结合实际模型理解  
`scan_en`：扫描移位使能  
`edt_update`：EDT 更新控制  
`test_clock`：扫描测试时钟源之一，后续用于形成内部测试时钟信号

把修改后的设计输出为第一阶段网表；`-replace` 允许覆盖已有同名文件

```tcl
write_design -output_file ./netlist/${top}.v -replace
```

输出 `./netlist/top.v`，作为第二步的 `read_verilog` 输入  
这条命令输出网表，没有在本步骤插入边界扫描单元、MBIST、EDT 或扫描链

### 模块六：第二步——声明芯片级需求，识别 JTAG 端口

指定当前设计层级为完整芯片

```tcl
set_design_level chip
```

声明本阶段需要边界扫描与存储器测试结构

```tcl
set_dft_specification_requirements -boundary_scan on -memory_test on
```

这时是在提出需求，不是已经完成了硬件插入  
后面的 `create_dft_specification` 建立配置，`process_dft_specification` 才处理并实现配置

把实际端口与工具识别的 JTAG 功能关联起来

```tcl
set_attribute_value TCK -name function -value tck
set_attribute_value TDI -name function -value tdi
set_attribute_value TMS -name function -value tms
set_attribute_value TRST -name function -value trst
set_attribute_value TDO -name function -value tdo
```

`-name` function 表示设置功能属性  
`-value` tck/tdi/... 表示该端口在 JTAG 接口中承担的角色  
这是给端口增加语义，不是把端口改名，也不是给端口赋逻辑 0 或 1  
工具据此识别 TAP 的外部接口，并在处理 DFT 配置时接入对应硬件

### 模块七：第二步——配置边界扫描端口、扫描使能和功能时钟

对两个功能时钟端口采用 clock 类型的边界扫描处理选项

```tcl
set_boundary_scan_port_options clk -cell_options clock
set_boundary_scan_port_options ramclk -cell_options clock
```

对这三个专用测试控制端口，不按普通功能 I/O 的方式插入边界扫描单元

```tcl
set_boundary_scan_port_options scan_en -cell_options dont_touch
set_boundary_scan_port_options edt_update -cell_options dont_touch
set_boundary_scan_port_options test_clock -cell_options dont_touch
```

这里的 `dont_touch` 是边界扫描端口的 `cell_options` 配置  
不要把它扩大解释为“整个设计中该端口相关逻辑永远不允许任何修改”

声明 `scan_en` 是工具认识的 DFT 信号，并使用已有节点作为信号源

```tcl
add_dft_signals scan_en -source_nodes scan_en
```

登记两个时钟：非激活值为 0，周期为 5ns

```tcl
add_clocks 0 clk -period 5ns
add_clocks 0 ramclk -period 5ns
```

第一个参数 0 是时钟的非激活电平，不是“时钟编号为 0”  
5ns 对应 200MHz 的周期描述，但不能据此断言后续 ATPG 的每个时序周期都是 5ns  
clk 与 ramclk 分别作为本实验的两个功能时钟来源

执行设计规则检查，为后续 DFT 配置创建和处理做准备

```tcl
check_design_rules
```

这条命令进行检查，不是生成测试向量；检查是否通过要看实际运行信息

### 模块八：第二步——创建 DFT Specification，并预留 EDT 可复用通道

根据前面的设计、时钟、端口属性和 DFT 需求创建配置对象  
把返回的配置对象路径/句柄保存到变量 spec 中

```tcl
set spec [create_dft_specification]
```

显示当前配置内容

```tcl
report_config_data $spec
```

在 BoundaryScan 配置下，增加辅助输入输出端口设置

```tcl
read_config_data -in ${spec}/BoundaryScan -from_string {
    AuxiliaryInputOutputPorts{
        auxiliary_input_ports : net_in[0], net_in[1] ;
        auxiliary_output_ports : net_out[0], net_out[1] ;
    }
}
```

`-in` 指明要修改的配置位置  
`-from_string` 表示配置内容直接写在后面的字符串中，而不是从文件读取  
大括号内是 Tessent 配置数据语法，不是普通 Tcl 命令列表  
`auxiliary_input_ports`：`net_in[0]`、`net_in[1]`  
`auxiliary_output_ports`：`net_out[0]`、`net_out[1]`  
这些辅助路径让功能 I/O 能在测试模式下作为测试通道使用  
第三步将把它们接到 EDT 的两个输入通道和两个输出通道  
这一步是在边界扫描结构中安排辅助路径，不是新建四个独立的顶层 EDT 端口

再次报告，以检查配置更新后的内容

```tcl
report_config_data $spec
```

处理 DFT 配置，生成并接入本阶段的测试结构

```tcl
process_dft_specification
```

提取 ICL，用于描述 IJTAG 测试访问结构、仪器接口及连接关系

```tcl
extract_icl
```

综合新增 DFT 仪器逻辑，使其映射到门级单元

```tcl
run_synthesis
```

第二步的三个动作可以记成：生成/接入硬件 → 提取访问描述 → 综合新增仪器  
ICL 是 Instrument Connectivity Language，重点是测试仪器连接与访问，不是 ATPG 测试数据  
本阶段在 TSDB 中使用 `design_id`=bscan，供第三步读取

补充核对：保存的 `top_bscan.dft_spec` 中，MemoryBist/Controller(c1) 配置了  
ramclk 时钟域，以及 m1、m2、m3 三个接口，对应 `dram_1`、`dram_2`、`dram_3`  
这些具体接口由生成的配置文件列出，不是主脚本手工逐条指定的

### 模块九：第三步——声明 EDT/OCC 所需的 DFT 信号

本步骤已通过 `read_design` top `-design_id` bscan `-verbose` 载入第二步设计

声明逻辑测试使能

```tcl
add_dft_signals ltest_en
```

使用已有节点提供扫描使能、EDT 更新和测试时钟

```tcl
add_dft_signals scan_en edt_update test_clock -source_node { scan_en edt_update test_clock }
```

要求工具根据其他 DFT 信号创建所需的内部测试时钟信号

```tcl
add_dft_signals shift_capture_clock edt_clock -create_from_other_signals
```

声明 EDT 模式、与 TCK 相关的测试时钟控制、存储器旁路控制信号

```tcl
add_dft_signals edt_mode
add_dft_signals tck_occ_en
add_dft_signals memory_bypass_en
```

`ltest_en`：逻辑测试使能，本实验也用它控制辅助 I/O 通道的使能端  
`scan_en`：控制扫描移位与捕获状态  
`edt_update`：EDT 相关更新控制  
`test_clock`：已有外部测试时钟源  
`shift_capture_clock`：供扫描移位/捕获过程使用的内部测试时钟信号  
`edt_clock`：EDT 使用的时钟信号  
`edt_mode`：EDT 相关测试模式控制信号  
`tck_occ_en`：与测试仪器的 TCK/OCC 测试时钟选择相关的控制信号  
`memory_bypass_en`：存储器旁路控制信号，便于逻辑扫描测试

`tck_occ_en` 不是 TCK 时钟本身；把它设为 1 不等于把 TCK 永久拉高  
声明 DFT 信号不一定会新增顶层引脚；工具可通过内部测试控制结构实现  
第二步原文使用 `-source_nodes`，第三步原文使用 `-source_node`，笔记分别保留原写法  
没有指定 `-source_node` 的声明，也不能直接理解成该信号没有来源  
具体实现位置和当前取值可由 `report_dft_signals`、生成的网表和测试描述核对

### 模块十：第三步——用 ltest_en 控制辅助输入输出路径

找到辅助输入路径的 enable 控制引脚，并把它与 `ltest_en` 建立控制关系

```tcl
add_dft_control_points [get_auxiliary_pins net_in[0] -enable -direction input] -dft_signal_source_name ltest_en
add_dft_control_points [get_auxiliary_pins net_in[1] -enable -direction input] -dft_signal_source_name ltest_en
```

辅助输出路径采用相同的逻辑测试使能控制

```tcl
add_dft_control_points [get_auxiliary_pins net_out[0] -enable -direction output] -dft_signal_source_name ltest_en
add_dft_control_points [get_auxiliary_pins net_out[1] -enable -direction output] -dft_signal_source_name ltest_en
```

`get_auxiliary_pins`：查询第二步形成的辅助引脚  
`-enable`：查询该辅助路径的使能控制引脚  
`-direction` input/output：指明辅助输入或辅助输出方向  
`-dft_signal_source_name` `ltest_en`：用名为 `ltest_en` 的 DFT 信号控制这些控制点  
这里连接的是“通道使能”，不是把 `ltest_en` 当作 EDT 的扫描数据输入  
后面 EDT/Connections 的 `port_pin_name` 才配置扫描数据通道接到哪里

声明本阶段要支持逻辑测试

```tcl
set_dft_specification_requirements -logic_test On
```

检查设计规则，并报告已安排的 DFT 控制点

```tcl
check_design_rules
report_dft_control_points
```

### 模块十一：第三步——建立 IJTAG 访问分支，配置两个 OCC 控制器

创建 DFT 配置，并为 occ、edt 两类扫描资源仪器安排 SIB 访问分支

```tcl
set spec [create_dft_specification -sri_sib_list {occ edt} ]
report_config_data $spec
```

SIB 是 Segment Insertion Bit，用于选择是否把相应测试分支接入访问链  
{occ edt} 中的名字用于后续配置引用，不是两个顶层端口名称

指定 OCC 使用名为 occ 的 SIB 作为 IJTAG 主机接口

```tcl
read_config_data -in $spec -from_string {
  OCC {
    ijtag_host_interface : Sib(occ);
  }
}
```

建立“控制器标识、要拦接的时钟节点”的成对列表

```tcl
set id_clk_list [list \
     clk clk \
     ramclk ramclk \
]
```

第一对：控制器 id=clk，时钟节点=clk  
第二对：控制器 id=ramclk，时钟节点=ramclk  
反斜线表示一条 Tcl 命令换到下一行继续书写

每次取出两个列表元素，创建一个 OCC 控制器并指定其时钟拦接节点

```tcl
foreach {id clk} $id_clk_list {
  set occ [add_config_element OCC/Controller($id) -in $spec]
  set_config_value clock_intercept_node -in $occ $clk
}
```

`add_config_element` 在配置对象中增加一个控制器条目，返回该条目的引用  
`set_config_value` 为该控制器指定 `clock_intercept_node`  
`clock_intercept_node` 表示 OCC 需要接入并控制的功能时钟路径节点  
此时仍在编辑配置，真正实现发生在后面的 `process_dft_specification`  
按原脚本注释，`scan_enable` 和 `shift_capture_clock` 会自动连接到 OCC 实例

报告加入 OCC 控制器后的配置

```tcl
report_config_data $spec
```

OCC 的用途是控制测试移位与捕获所需的时钟路径/脉冲  
插入 OCC 不等于本脚本已经开展 transition 或 at-speed ATPG

### 模块十二：第三步——配置 EDT 的链数、外部通道数和 IJTAG 接口

在同一份 DFT 配置中增加 EDT 设置

```tcl
read_config_data -in $spec -from_string {
   EDT {
     ijtag_host_interface : Sib(edt);
       Controller (c1) {
        longest_chain_range : 2, 100;
        scan_chain_count : 6;
        input_channel_count : 2;
        output_channel_count : 2;
        Connections +{
            EdtChannelsIn(1) {
            }
            EdtChannelsIn(2) {
            }
            EdtChannelsOut(1) {
            }
            EdtChannelsOut(2) {
            }
          }
       }
    }
}
```

`ijtag_host_interface` : Sib(edt)：EDT 控制器从 edt 这条 IJTAG 分支访问  
Controller(c1)：建立名为 c1 的 EDT 控制器配置  
`longest_chain_range` : 2, 100：EDT 生成时支持的“最长内部扫描链长度”的范围  
它不是“每条链都固定为 100”，也不能据此写出最终扫描链的实际长度  
`scan_chain_count` : 6：配置 6 条内部扫描链  
`input_channel_count` : 2：配置 2 个外部 EDT 输入通道  
`output_channel_count` : 2：配置 2 个外部 EDT 输出通道  
Connections +{ ... }：按该配置语法补充连接子项，预先建立四个通道连接配置

数据方向可以理解为：  
2 个外部输入通道 → EDT 解压逻辑 → 6 条内部扫描链  
6 条内部扫描链 → EDT 响应压缩逻辑 → 2 个外部输出通道  
6/2=3 只是链数和通道数的比例，不能直接当成最终测试时间或数据量压缩倍数

原脚本还说明：EDT 控制器带有 bypass 能力  
边界扫描段连接到低编号内部链的行为涉及 `connect_bscan_segments_to_lsb_chains` 的自动设置  
该属性没有在当前脚本中显式赋值；这里不把注释改写成额外执行过的命令

### 模块十三：第三步——把 EDT 通道接到已有辅助引脚，并实现设计

EDT 输入通道 1 使用 `net_in[0]` 的辅助输入数据引脚

```tcl
set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsIn(1) [get_single_name [get_auxiliary_pins net_in[0] -direction input]]
```

EDT 输入通道 2 使用 `net_in[1]` 的辅助输入数据引脚

```tcl
set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsIn(2) [get_single_name [get_auxiliary_pins net_in[1] -direction input]]
```

EDT 输出通道 1 使用 `net_out[0]` 的辅助输出数据引脚

```tcl
set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsOut(1) [get_single_name [get_auxiliary_pins net_out[0] -direction output] ]
```

EDT 输出通道 2 使用 `net_out[1]` 的辅助输出数据引脚

```tcl
set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsOut(2) [get_single_name [get_auxiliary_pins net_out[1] -direction output] ]
```

这里没有 `-enable`，获取的是辅助数据路径引脚；与模块十查询使能引脚的用途不同  
`get_single_name`：将查询得到的单个对象转换为名称，供 `port_pin_name` 属性使用  
`port_pin_name`：给该 EDT 通道指定实际连接的端口/引脚名称  
不硬编码生成实例的完整名称，而是通过 `get_auxiliary_pins` 查询，更便于随设计更新

报告最终配置，再处理并实现新增硬件

```tcl
report_config_data $spec
process_dft_specification
extract_icl
run_synthesis
```

本步骤输出 `design_id`=edt 的设计，供第四步读取  
EDT/OCC 已加入，并不等于功能寄存器已经全部连成扫描链  
内部链的分析和实际插入在第四步执行

### 模块十四：第四步——进入分析模式，识别 EDT 实例并建立扫描模式

读取 `design_id`=edt 后，切换到扫描分析模式

```tcl
set_system_mode analysis
```

报告设计中已有的扫描段以及工具识别到的时钟

```tcl
report_scan_segments
report_clocks
```

扫描段可能来自之前已插入的测试结构；不能把“已有扫描段”理解成“完整扫描链已插好”

逐层查询实际 EDT 实例名，并保存到 `edt_instance`

```tcl
set edt_instance [get_name_list [get_instance -of_module [get_name [get_icl_module -filter tessent_instrument_type==mentor::edt]]]]
```

由内向外理解这条嵌套查询：
1. `get_icl_module` `-filter` `tessent_instrument_type`==mentor::edt
  在 ICL 模块中筛选类型为 mentor::edt 的仪器
2. `get_name`
  获取对应模块名称
3. `get_instance` `-of_module` ...
  找到设计中属于该模块的实例
4. `get_name_list`
  将实例对象转换为名称列表，保存到变量 `edt_instance`
这种写法避免手工填写工具生成的 EDT 实例长名字
该脚本按本实验的 EDT 配置使用；复杂多控制器设计需要重新确认查询和模式安排

建立名为 `edt_mode` 的扫描模式，并关联相应 EDT 实例

```tcl
add_scan_mode edt_mode -edt_instance $edt_instance
```

这里 `edt_mode` 是扫描模式名称  
第三步 `add_dft_signals` `edt_mode` 声明的是同名 DFT 控制信号，二者所属对象不同  
建立扫描模式是在规定链和 EDT 的组织关系，不是生成 ATPG 向量

### 模块十五：第四步——分析扫描链并真正插入测试逻辑

根据当前扫描模式、EDT 配置、时钟和已有扫描段分析链分配

```tcl
analyze_scan_chains
```

显示链分配分析结果

```tcl
report_scan_chains
```

真正执行扫描测试逻辑的插入和连接

```tcl
insert_test_logic
```

可以依次记成：分析怎么连 → 报告计划/分析结果 → 实际插入和连接  
本阶段的目标是让可扫描时序单元及需要纳入的已有扫描段形成测试移位路径  
6 条内部扫描链的数量配置来自第三步，而不是本脚本又手工写了一次链数  
原脚本中的 `report_scan_chains` 在 `insert_test_logic` 之前，不是插入后的再次验证报告  
第四步上下文使用 `design_id`=scan，第五步从该版本继续  
不能仅凭这些指令给出最终链长、寄存器数量或 DRC 全部通过的结论

### 模块十六：第五步——设置 ATPG 会话模式并导入扫描模式

当前上下文为 patterns `-scan`，已读取 `design_id`=scan 的设计

选择当前 ATPG 会话/结果的模式名

```tcl
set_current_mode edt_stuck
```

报告已识别的 DFT 信号

```tcl
report_dft_signals
```

导入第四步已经建立的扫描模式

```tcl
import_scan_mode edt_mode
```

报告核心测试描述、时钟和 DFT 信号

```tcl
report_core_descriptions
report_clocks
report_dft_signals
```

三个名字要分清：  
scan：设计检查点的 `design_id`，由 `read_design` 选择  
`edt_mode`：扫描模式，由 `add_scan_mode` 建立、由 `import_scan_mode` 导入  
`edt_stuck`：当前 ATPG 会话/结果模式名，由 `set_current_mode` 选择

`edt_stuck` 的命名表示这个示例面向 EDT stuck-at 测试  
但 `set_current_mode` 的参数是模式名，不能把这个名字解释为显式设置故障模型的命令  
当前脚本没有 `set_fault_type` transition 等切换到 transition 模型的命令  
已有 Verilog 测试平台头部标记 Fault : STUCK，可辅助核对当前材料中的向量类型

### 模块十七：第五步——设置静态 DFT 控制值，生成向量并报告

将 `tck_occ_en` 的静态测试控制值设为 1

```tcl
set_static_dft_signal_values tck_occ_en 1
```

这个设置用于本次测试过程中的相关时钟控制，不是给顶层 TCK 施加一个固定高电平  
本脚本只在这里显式设置 `tck_occ_en`，其他信号的设置应结合导入模式和报告判断

报告当前静态 DFT 信号配置

```tcl
report_static_dft_signal_settings
```

进入 ATPG 分析模式

```tcl
set_system_mode analysis
```

根据当前故障模型、扫描模式和测试约束创建测试向量

```tcl
create_patterns
```

输出统计及更详细的分析信息

```tcl
report_statistics -detailed_analysis
```

报告扫描测试数据量等信息

```tcl
report_scan_volume
```

`create_patterns` 才是实际生成测试向量的核心命令  
`report_statistics`、`report_scan_volume` 只是报告，不会再插入一遍 EDT 或扫描链  
覆盖率、故障分类数量和 pattern 数量需要实际统计或相应产物才能确认  
本笔记按用户要求解释指令，不填写从 .log 文件提取的数值

### 模块十八：第五步——导出 STIL、并行 Verilog 和抽样串行 Verilog

首先导出 STIL 格式向量

```tcl
write_patterns patterns/pattern.stil -stil -replace
```

STIL 是 Standard Test Interface Language，用来描述测试信号、时序、过程和向量  
`-stil` 指定导出格式；`-replace` 允许覆盖同名输出  
实际脚本输出路径是 patterns/`pattern.stil`，不是 flow 根目录下的 `pattern.stil`  
目录里即便另有同名文件，也不能仅凭文件名就认定其内容和来源完全一致

随后导出并行 Verilog 测试平台及配套向量数据

```tcl
write_patterns patterns/top_stuck_parallel.v -verilog -parallel -replace -scan -parameter_list {SIM_KEEP_PATH 1}
```

`-verilog`：输出可供仿真使用的 Verilog 测试平台形式  
`-parallel`：采用并行扫描装载/卸载的仿真形式，用于减少扫描移位仿真的开销  
`-scan`：指定扫描测试相关的向量输出；按本示例与 `-parallel` 一起使用  
并行仿真形式不能等同于芯片引脚上真实执行逐拍串行移位的过程  
`-parameter_list` {`SIM_KEEP_PATH` 1}：让生成的测试平台保留配套 pattern 数据文件的路径信息  
它影响测试平台查找数据文件的方式，不是保持 DUT 逻辑层次的综合选项  
保留相对路径仍依赖仿真工作目录，不会自动变成绝对路径

给后续的向量导出设置抽样过滤条件：每种类型抽取 2 个

```tcl
set_pattern_filtering -sample_per_type 2
```

导出串行 Verilog 测试平台及配套数据文件

```tcl
write_patterns patterns/top_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}
```

`-serial`：按串行移位方式表示扫描测试，能检查相关串行移位行为  
由于抽样过滤设置在这条命令之前，串行输出使用该抽样条件  
前面的 STIL 和并行 Verilog 已经写出，不会被后面的过滤命令追溯修改  
`-sample_per_type` 2 不是“所有向量总共只剩两个”，也不是重新执行 ATPG  
三种输出的用途不同，不应只比较文件大小就推断故障覆盖率高低

原文件末尾还有下面这一行，但它前面有 #，因此不会执行

```tcl
# write_tsdb_data -replace
```

`write_tsdb_data` 用于显式写出相应 TSDB 数据  
当前行被注释，不能把它当作本脚本已经执行的一条保存命令  
目录中存在历史数据库文件，也不能据此认定就是这条被注释的命令生成的

### 模块十九：综合辅助指令——.synopsys_dc.setup 的作用

以下两条是 flow/.`synopsys_dc.setup` 的原始内容，在 DC 环境使用

```tcl
set link_library /home/gipsydanger/Desktop/tessent_learning/ref_exp1/tessent/flat_flow_netlist/lib/adk.db
set target_library /home/gipsydanger/Desktop/tessent_learning/ref_exp1/tessent/flat_flow_netlist/lib/adk.db
```

`link_library`：DC 链接设计引用时使用的库  
`target_library`：DC 综合映射时使用的目标工艺库  
本例两者都指向 `adk.db`  
`adk.db` 是 DC 使用的综合库，前面的 `adk.tcelllib` 是 Tessent 使用的测试单元库  
两者不能仅改扩展名互相替代

这两个 /workspace/... 路径来自原来的 Linux 环境  
当前 E: 盘材料的路径不会自动替换这些历史绝对路径  
若要在另一套 EDA 环境复现，应把库路径和工具环境改成那套环境中真实存在的位置  
本笔记保留原始指令用于理解，没有修改源文件

### 模块二十：综合辅助指令——run_synthesis 背后的脚本分工

第二、三步的 `run_synthesis` 对应以下两个综合工作目录：

```text
synthesis_outdir/top_bscan.synthesis_dc_shell/
synthesis_outdir/top_edt.synthesis_dc_shell/
```

主要文件分工如下：  
`setup.script`：建立 PATH、工具目录、动态库及许可证等环境变量  
`synthesis.script`：调用 DC、检查预期输出文件是否存在  
`dc_shell.synthesis_tcl`：DC 实际执行的读取、综合、输出命令  
`file_list`：列出需要回写 TSDB 的综合结果文件  
`file_copy.script`：按清单检查并回写综合后的仪器网表  
.`synopsys_dc.setup`：配置 DC 的 `link_library`、`target_library`  
这些文件属于生成的综合辅助流程，不是新的第六、第七个 DFT 主步骤

`synthesis.script` 中先加载环境设置

```bash
if [ -f ./setup.script ] ; then
  . ./setup.script
fi
```

这是 shell 语法，不是 Tessent Tcl  
. `./setup.script` 表示在当前 shell 中加载环境变量

随后调用 DC 执行综合 Tcl

```bash
/workspace/eda_tools/syn_vR-2020.09-SP1/bin/dc_shell -f dc_shell.synthesis_tcl
```

两个 `synthesis.script` 都会清理各自用于接收本次输出的仪器子目录  
随后按 `file_list` 检查标记为 synth 的输出文件是否非空  
若输出文件缺失，脚本会报告错误并以非零状态退出  
`file_copy.script` 再按清单把对应 .vg 仪器网表复制回 `../../tsdb/instruments`  
这些是原辅助脚本的行为说明，本次只阅读，没有执行清理、综合或复制操作

DC Tcl 中定义了 `compile_local` 辅助过程，核心内容如下

```tcl
proc compile_local {args} {
    set_scan_configuration -style multiplexed_flip_flop
    eval compile $args
    change_names -rules verilog -hier
}
```

`set_scan_configuration`：指定扫描触发器相关的综合风格  
eval compile `$args`：把传入的可选参数交给 compile，执行逻辑综合/映射  
`change_names`：把层次内名称调整为符合 Verilog 的命名规则  
该过程的定义受外层“尚未存在同名命令”的条件保护，这里摘录其主体讲解

以第二步 TAP 模块的综合为例，原脚本包含以下命令

```tcl
current_design top_bscan_tessent_tap_main
compile_local
write -f verilog -output top_bscan_ijtag.instrument/top_bscan_tessent_tap_main.vg \
  [check_modules { \
    top_bscan_tessent_tap_main \
    top_bscan_tessent_tap_main_fsm \
  }]
set_dont_touch [get_designs top_bscan_tessent_tap_main]
```

在执行上述片段前，原脚本已用 `read_file` 读取相应仪器的 Verilog 源  
`current_design` 选择当前综合的仪器模块  
`compile_local` 调用前面定义的综合过程  
`check_modules` 是本文件定义的辅助过程，检查和整理要输出的模块列表  
write `-f` verilog 输出 .vg 门级仪器网表  
`set_dont_touch` 对已处理的设计设置保护，避免后续步骤再对它随意优化  
原文在适用位置还会对 `tessent_persistent_cell_*` 实例设置 `set_size_only`  
并为部分模块使用 `compile_local` `-boundary_optimization`

MBIST 控制器组合模块还会加载它自己的 SDC，并执行相应约束辅助过程

```tcl
source ../../tsdb/instruments/top_bscan_mbist.instrument/top_bscan_tessent_mbist_c1_controller_assembly.sdc
tessent_set_default_variables
tessent_create_functional_clocks
tessent_constrain_top_bscan_tessent_mbist_c1_controller_assembly_non_modal
```

这部分给 MBIST 仪器综合提供变量、功能时钟和非模态约束  
DC 文件中的 Tcl、生成的 SDC 与 Tessent 主脚本在不同工具/阶段中使用，应分清执行环境

第二步综合文件集中在 TAP、IJTAG SIB、边界扫描单元及 MBIST 控制器/接口  
第三步集中在新增 IJTAG SIB/TDR、两个 OCC 模块及 EDT c1 模块  
因此本实验虽从门级网表开始，仍需综合新增仪器的 RTL  
不能把这两次 `run_synthesis` 理解成 `flat_flow_rtl` 中那种从功能 RTL 开始的全设计综合

### 模块二十一：已有验证配置的含义——与五个主脚本区分

目录中还保留了 tsdb/patterns/`top_bscan.patterns_spec_signoff`  
它是一份仪器验证向量配置，不是 Tcl 主流程脚本，也不是运行日志  
其中列出了以下几类验证任务：  
ICLNetwork：核对仪器连接/访问网络  
JtagBscanPatterns：JTAG 与边界扫描的验证操作  
`MemoryBist_P1`：MBIST 相关操作  
`MemoryBist_ParallelRetentionTest_P1`：存储器保持测试相关操作

该配置中的几个属性示例

```text
ConstantPortSettings {
  scan_en : 0;
}
```

表示该验证配置将 `scan_en` 固定为 0  
它不是第五步所有 ATPG 扫描过程的统一扫描使能值

```text
ClockPeriods {
  ramclk : 5.0ns;
}
```

表示相应 MBIST 验证任务的 ramclk 周期设置

```text
reduced_address_count : on;
```

表示相应 MBIST 验证任务采用减少地址数量的设置，便于验证过程  
不能将这种验证配置直接等同于完整容量的生产测试已经通过

五个主脚本中没有显式写出 `create_patterns_specification`、`process_patterns_specification`  
或 `run_testbench_simulations` 等验证调用  
因此目录中有验证配置、Verilog 测试平台和 `simulation_outdir`，不代表当前五个脚本  
已经逐项调用了全部仪器验证流程，更不能仅凭文件存在判断仿真通过

patterns 目录下的 .v、.vec、.cfg、.`chain.name`、.`po.name` 是配套测试产物  
.v 是测试平台，.vec 保存数据，其他文件记录配套数据/名称等信息  
阅读主指令时应理解它们的用途，不必把海量逐拍向量当成新的 DFT 插入命令来解释

### 模块二十二：把几个相近概念分开记忆

#### PAD 与 Boundary Scan

PAD 提供本实验中的芯片输入/输出单元模型；第一步处理  
Boundary Scan 在 I/O 边界提供测试控制与观察结构；第二步处理  
插入 PAD 不等于插入边界扫描寄存器

#### TAP 与 IJTAG

TAP 提供 JTAG 外部访问控制接口  
IJTAG 用 SIB/TDR 等结构把访问延伸到芯片内部测试仪器  
JTAG 的 TDI/TDO 访问路径不能直接等同于本实验的两对 EDT 数据通道

#### MBIST 与逻辑 ATPG

MBIST 面向存储器内部测试，依赖存储器描述、控制器和接口  
逻辑 ATPG 面向当前逻辑故障模型生成扫描测试向量  
插入 MBIST 硬件不等于第五步 `create_patterns` 自动完成了全部存储器测试

#### EDT 与 Scan Chain

EDT 负责扫描数据解压和响应压缩，第三步插入  
Scan Chain 是内部扫描移位路径，第四步分析并连接  
EDT 的外部通道数与内部扫描链数是不同的数量

#### OCC 与 at-speed ATPG

OCC 是测试时钟控制硬件  
是否执行 transition/at-speed 测试，还取决于故障模型、测试协议和 ATPG 配置  
仅看到 OCC 已插入，不能判断已经进行了 at-speed 测试

#### create_dft_specification、process_dft_specification、create_patterns

`create_dft_specification`：建立 DFT 硬件配置  
`process_dft_specification`：按配置生成并集成 DFT 硬件  
`create_patterns`：针对当前扫描测试设计生成测试向量  
三者分别对应“建配置”“做硬件”“做测试数据”

#### set_context 与 set_system_mode

`set_context` 决定本次做哪一大类工作，例如 dft 或 patterns  
`set_system_mode` 决定当前上下文中的处理状态，例如 insertion 或 analysis  
同样写 analysis，第四步是在扫描分析上下文，第五步是在向量生成上下文

### 模块二十三：按原文件顺序回顾每一步

#### 一、`1_insert_pad`

```text
初始化与读库 → 读原始门级网表 → set_current_design
→ insertion → 为原有输入插 ipad → 为原有输出插 opad
→ 创建测试端口和对应 PAD → write_design
```

#### 二、`2_insert_bscan&mbist`

```text
初始化与读库 → 读带 PAD 网表 → 设 chip 层级与 Boundary Scan/Memory Test 需求
→ JTAG 端口功能标记 → 边界扫描端口选项 → scan_en 和时钟声明
→ DRC → 创建配置 → 加辅助 I/O 配置 → process_dft_specification
→ extract_icl → run_synthesis
```

#### 三、`3_insert_edt_occ`

```text
初始化与读库 → 读 bscan 检查点 → 声明测试信号
→ 辅助通道使能控制 → logic_test 需求 → DRC
→ 创建含 occ/edt SIB 的配置 → 两个 OCC 控制器 → EDT 参数
→ EDT 通道绑定 → process_dft_specification → extract_icl → run_synthesis
```

#### 四、`4_insert_scan`

```text
初始化与读库 → 读 edt 检查点 → analysis → 报告已有扫描段和时钟
→ 查询 EDT 实例 → add_scan_mode → analyze_scan_chains
→ report_scan_chains → insert_test_logic
```

#### 五、`5_atpg`

```text
初始化与读库 → 读 scan 检查点 → 设置 edt_stuck 会话模式
→ import_scan_mode edt_mode → 查看测试描述/时钟/信号
→ tck_occ_en=1 → analysis → create_patterns → 报告统计和扫描数据量
→ 导出 STIL → 导出并行 Verilog → 设置抽样过滤 → 导出串行 Verilog
```

阅读顺序建议：先看每步读入哪个版本，再看新增什么配置/硬件，最后看输出到哪里  
五步之间依靠门级网表和 TSDB 设计版本交接，而不是只靠文件编号本身建立依赖

## 4. 参考文件（均为非 .log 文件）

格式参考：`C:\Users\zxk__\Desktop\tessent笔记.txt`

### 主指令

```text
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\1_insert_pad
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\2_insert_bscan&mbist
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\3_insert_edt_occ
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\4_insert_scan
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\5_atpg
E:\tessent\Reference_2026\tessent\flat_flow_netlist\flow\.synopsys_dc.setup
```

### 综合辅助指令

```text
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\setup.script
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\synthesis.script
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\dc_shell.synthesis_tcl
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\file_copy.script
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\file_list
flow\synthesis_outdir\top_bscan.synthesis_dc_shell\.synopsys_dc.setup
flow\synthesis_outdir\top_edt.synthesis_dc_shell\setup.script
flow\synthesis_outdir\top_edt.synthesis_dc_shell\synthesis.script
flow\synthesis_outdir\top_edt.synthesis_dc_shell\dc_shell.synthesis_tcl
flow\synthesis_outdir\top_edt.synthesis_dc_shell\file_copy.script
flow\synthesis_outdir\top_edt.synthesis_dc_shell\file_list
flow\synthesis_outdir\top_edt.synthesis_dc_shell\.synopsys_dc.setup
```

### 辅助核对的配置/产物

```text
flow\tsdb\dft_inserted_designs\top_bscan.dft_spec
flow\tsdb\dft_inserted_designs\top_edt.dft_spec
flow\tsdb\patterns\top_bscan.patterns_spec_signoff
flow\patterns\top_stuck_parallel.v（文件头部）
flow\patterns\top_stuck_parallel.v.cfg
flow\patterns\top_stuck_serial.v.cfg
```

以上 flow\... 均相对于 `E:\tessent\Reference_2026\tessent\flat_flow_netlist`
