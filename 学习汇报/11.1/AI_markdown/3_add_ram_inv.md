# 3_add_ram_inv 实验流程与命令详解

实验目录：[3_add_ram_inv](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv)；顶层模块：`add_ram_inv`。

本说明按“实验目的 → 文件与输入输出 → 执行顺序 → 逐命令解释 → 结果解读 → 差异和问题 → 复现检查”的结构编写。脚本代表配置意图，日志或产物代表历史证据；没有证据的结果不作推定。本次仅阅读和核对文件，没有重新执行 Tessent、Design Compiler 或仿真器，也没有修改 E 盘的实验源文件。

文中的 Bash 命令用于安装了 EDA 工具的 Linux 环境；E 盘是当前 Windows 上的材料位置。`dofile` 后的脚本在 Tessent 中执行，`.proc` 是测试过程描述，不是普通 Tcl。迁移路径必须按原有目录相对关系处理。

本实验是 hier_flow 的可独立运行子模块流程，历史工具为 Tessent 2021.3。它先生成本核心的测试结构、graybox 和内部/外部向量，再把结果交给顶层复用。总流程见 [hier_flow 顶层说明](C:/Users/zxk__/Desktop/GitHub/markdown/hier_flow.md)。

## 1. 这个实验要完成什么

对物理块 add_ram_inv 完成 MBIST、EDT/OCC、wrapper 和扫描插入，建立两个扫描模式：内部测试模式测核心逻辑，外部模式用 wrapper 控制/观察块边界。之后生成 graybox，分别跑 external 和 internal ATPG，最后合并两种模式的故障检测结果。

```text
netlist/*.v（add_ram_inv + adder）
  → 1_insert_mbist → design_id=mbist
  → 2_insert_edt&occ → design_id=edt
  → 3_insert_scan → design_id=scan，int_mode + ext_mode
  → 4_gen_graybox → .vg_graybox + *_gray_box.tcd
  ├→ 5_ext_atpg（graybox，ext_mode）→ ext_multi_stuck
  └→ 6_internal_atpg（完整网表，int_mode）→ int_mode_sa
            ↓ read_faults -merge
       合并覆盖率统计（不等同于自动合并向量）
```

## 2. 文件、设计结构和产物

| 文件 | 阅读用途 |
|---|---|
| [1_insert_mbist](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/1_insert_mbist) | 物理块 MBIST 插入 |
| [2_insert_edt&occ](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/2_insert_edt&occ) | EDT/OCC 和层次化控制信号 |
| [3_insert_scan](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/3_insert_scan) | wrapper 分析、内外扫描模式和链连接 |
| [4_gen_graybox](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/4_gen_graybox) | 逐模式校验并生成外部模式 graybox |
| [5_ext_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/5_ext_atpg) | graybox external ATPG |
| [6_internal_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/6_internal_atpg) | 完整设计 internal ATPG 和故障结果合并 |

目录还保留上述各步日志、netlist、synthesis_outdir、patterns 和 TSDB；本案例不是只有模板。关键现存输出为：

- [扫描全网表](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/tsdb/dft_inserted_designs/add_ram_inv_scan.dft_inserted_design/add_ram_inv.vg)；
- [graybox 网表](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/tsdb/dft_inserted_designs/add_ram_inv_scan.dft_inserted_design/add_ram_inv.vg_graybox)；
- [独立 graybox TCD 描述](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/add_ram_inv_gray_box.tcd)；
- patterns 中的 internal/external 串行、并行 testbench 及 vec/cfg；
- TSDB 的 ext_multi_stuck 和 int_mode_sa 模式数据库。

功能结构包括 3 个 picdram、2 个 adder、16 个显式功能 dff。此设计采用两个 8 位输入总线 net_in_1、net_in_2；16 个功能 dff 使用 QB 反相输出。每个 picdram 的配套测试描述为 SRAM、70×8、同步 1RW、SMarchCHKB，不能按 7 位地址总线直接推成 128 字深度。

顶层将以实例 `add_ram_inv_1` 使用该核心，并以 `retargeting2_mode` 选择它的测试通道。另一个核心的说明见 [2_add_ram.md](C:/Users/zxk__/Desktop/GitHub/markdown/2_add_ram.md)；两份脚本框架类似，但输入接口、数据极性及 ATPG 数量不同。

## 3. 基础命令与模式名称

| 命令/名称 | 含义 |
|---|---|
| `set top add_ram_inv` | 定义模块名变量 |
| `set_context dft -no_rtl -design_id mbist/edt` | 门级仪器插入的不同阶段 |
| `set_context dft -no_rtl -scan -design_id scan` | 扫描插入上下文 |
| `set_tsdb_output_directory ./tsdb` | 数据库相对当前子实验目录保存 |
| `read_cell_library ../../lib/...` | 两级上方 hier_flow/lib 中的 DFT 模型 |
| `read_verilog ./netlist/*.v` | 第一步直接读入本核心原始门级源文件及 adder 依赖 |
| `set_current_design $top` | 将本核心设为当前顶层，展开并关联测试模型 |
| `set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory` | 存储器测试描述搜索路径 |
| `read_design ... -design_id ...` | 恢复某阶段设计和测试元数据 |
| int_mode / ext_mode | 扫描插入时定义的配置模式，也是对应 DFT 模式控制功能 |
| int_mode_sa / ext_multi_stuck | ATPG 输出模式名，与扫描配置名分开 |

int_mode/ext_mode 不能同时作为有效测试模式使用；int_ltest_en/ext_ltest_en 是相应测试控制功能，不是把名称当成一个可以互换的普通 scan_en。

## 4. 第一步：1_insert_mbist

**输入：** 本子目录 netlist/*.v；**输出：** mbist 阶段网表和仪器。

| 命令 | 作用 |
|---|---|
| `set_design_level physical_block` | 按物理块设计处理，未来在上层作为核心集成；本步不创建芯片级 PAD/TAP 接口 |
| `set_dft_specification_requirements -boundary_scan off -memory_test on -logic_test off` | 这一阶段只要求存储器测试，不启用芯片边界扫描与逻辑测试仪器 |
| `report_memory_instances` | 确认识别出的存储器实例 |
| `add_clocks 0 clk -period 5ns` | 声明功能 clk，关闭状态为 0 |
| `add_clocks 0 ramclk -period 5ns` | 声明 RAM 时钟 |
| `check_design_rules` | 检查可插入条件 |
| `create_dft_specification` | 根据设计及存储器模型生成配置 |
| `report_config_data` | 核对控制器、时钟域和 MemoryInterface |
| `process_dft_specification` | 生成/插入硬件 |
| `extract_icl` | 提取测试访问网络及配套数据 |
| `run_synthesis` | 用 DC 映射本步新增测试 RTL，并回填 TSDB |

历史 MBIST 配置为 1 个 Controller(c1)，时钟域 ramclk，MemoryInterface(m1/m2/m3) 连接 dram_1、dram_2、dram_3。综合记录 `pass 1 fail 0`。脚本中第 15 行 tck_occ_en 被注释，其声明在第 2 步完成。

## 5. 第二步：2_insert_edt&occ

### 5.1 控制信号

在读取 mbist 阶段后声明：

| 信号 | 作用 |
|---|---|
| ltest_en | 通用逻辑测试控制 |
| scan_en、edt_update、test_clock | 指定对应源节点，提供动态扫描/更新/时钟控制 |
| shift_capture_clock、edt_clock | 从其他 DFT 信号创建内部测试时钟 |
| memory_bypass_en | 逻辑测试时旁路存储器 |
| tck_occ_en | 控制 scan-tested 仪器相关捕获时钟 |
| int_mode、ext_mode | 定义可被扫描/ATPG流程识别的内外模式功能 |
| int_ltest_en、ext_ltest_en | 配合 wrapper 的测试控制功能 |

`add_dft_signals` 主要声明功能与来源，实际静态控制的实现和驱动由后续规格处理及 ICL 集成完成，不代表每个名称都要单独新加外部 PAD。

### 5.2 OCC/EDT 规格

```tcl
set_dft_specification_requirements -logic_test on
check_design_rules
set spec [create_dft_specification -sri_sib_list {edt occ}]
```

`read_config_data -in $spec -from_string` 向配置增加 OCC 和 EDT。通过 `foreach {id clk}`、`add_config_element`、`set_config_value clock_intercept_node`，为 clk 和 ramclk 各建立一个 OCC。

| EDT 参数 | 值 |
|---|---|
| longest_chain_range | 2～100 |
| scan_chain_count | 6 |
| input_channel_count | 2 |
| output_channel_count | 2 |
| ijtag_host_interface | Sib(edt) |

这里没有像芯片顶层那样指定 AuxiliaryInputOutputPorts；工具创建核心测试接口，供上层通过 TAM/modal connections 连接。`process_dft_specification → extract_icl → run_synthesis` 生成并映射本阶段仪器，历史综合也为 `pass 1 fail 0`。

## 6. 第三步：3_insert_scan

### 6.1 wrapper 识别

```tcl
set_wrapper_analysis_options -exclude_ports [get_ports {*edt_channels*}]
analyze_wrapper_cells
report_wrapper_cells -verbose
```

| 命令 | 作用 |
|---|---|
| `get_ports {*edt_channels*}` | 找出 EDT 通道端口 |
| `-exclude_ports` | 将这些测试数据接口排除在普通 wrapper 分析之外，避免错误包裹通道 |
| `analyze_wrapper_cells` | 识别可复用的共享 wrapper 与需新插入的专用 wrapper |
| `report_wrapper_cells -verbose` | 查看每个端口的 wrapper 类型、实例和时钟 |

wrapper 服务于核心隔离和边界控制/观察；它和芯片 PAD 边界扫描不是同一个结构层次。报告中既有共享的功能寄存器，也有专用输入/输出逻辑，实际实现不能简单按端口位数逐一相加推算。

### 6.2 两种扫描模式

```tcl
set edt_instance [get_instance -of_icl_instances [get_icl_instances -filter tessent_instrument_type==mentor::edt]]
add_scan_mode int_mode -edt_instance $edt_instance
add_scan_mode ext_mode -chain_count 2
analyze_scan_chains
report_scan_chains
insert_test_logic
report_scan_chains
```

内层 `get_icl_instances` 查找 EDT 仪器，`get_instance -of_icl_instances` 映射成设计实例。内部模式关联 EDT 的 6 条链，外部模式请求 2 条 wrapper 链。

历史结果：

| 模式 | 链 | 长度/单元数 |
|---|---|---|
| int_mode | 6 条 EDT 内部链 | 每条 46，共 276 |
| ext_mode | mixed wrapper chain0 | 20 |
| ext_mode | input wrapper chain1 | 21 |
| ext_mode 合计 | 2 条 | 41 |

插入汇总：新增顶层端口 2、实例 172、clock gater control 1、input wrapper logic 31、output wrapper logic 16、pipelining 3、retiming 12。不同类别可能不是可直接求和的互斥集合；以报告各自定义为准。

脚本末尾手工 write_design/write_atpg_setup 被注释，但 TSDB 扫描设计实际由 insert_test_logic 流程保存，不意味着没有结果。

## 7. 第四步：4_gen_graybox

graybox 保留上层测试所需的 wrapper、边界连接和测试资源，去除无需在上层完整建模的内部逻辑。它不等于完全空白 blackbox。

| 命令 | 作用 |
|---|---|
| `get_config_elements Core($top)/Scan/Mode -part tcd -silent` | 枚举 TCD 中的扫描模式配置 |
| `foreach_in_collection` | 对每个模式逐一处理 |
| `get_config_value $mode_wrapper -id <0>` | 取得模式标识名 |
| `get_config_value type -in $mode_wrapper` | 取得 internal/external 类型 |
| `import_scan_mode $mode_name` | 载入该模式 |
| `check_design_rules` | 逐模式验证扫描配置 |
| `if {$mode_type eq "external"}` | 只在 external 模式分支生成 graybox |
| `report_scan_cells` | 检查当前 external 模式的扫描单元 |
| `analyze_graybox` | 分析必须保留的设计实例 |
| `write_design -tsdb -graybox -verbose` | 将 graybox 网表和对应信息写入 TSDB |
| `write_core_description ... -replace` | 另写独立 TCD 描述 |
| `set_system_mode setup` | 每个模式处理完成后回 setup，为下一个模式重新配置 |

历史报告选择 **723 / 2290 个实例（31.57%）**进入 graybox。其中 488 个来自 ICL scan resource instruments，3 个来自保留连接逻辑，232 个来自 wrapper 和边界相关逻辑。记录可以解释缩减范围，但不能将百分比直接当作综合面积节省。

## 8. 第五步：5_ext_atpg

**输入：** `read_design ... -view graybox`；**用途：** 测试核心外部模式可见的 wrapper/边界逻辑。

| 命令 | 作用 |
|---|---|
| `set_current_mode ext_multi_stuck -type external` | 创建 external 类型 ATPG 结果模式 |
| `import_scan_mode ext_mode` | 导入第 3 步的 2 链 wrapper 配置 |
| `report_core_instances` / `report_static_dft_signal_settings` | 核对核心描述与静态模式 |
| `add_black_boxes -auto` | 自动添加分析所需的 blackbox 模型处理，配合 graybox 省略内部实现 |
| `set_system_mode analysis` | 建立 external 分析模型 |
| `set_xclock_handling x` | 按 X 方式处理未知时钟情况，保持分析语义；不是把未知时钟当作确定 0 |
| `create_patterns` | 生成本模式测试 |
| `write_tsdb_data -replace` | 在过滤前保存完整模式数据 |
| `write_patterns ...parallel.v -verilog -parallel ...` | 先导出完整并行测试平台 |
| `set_pattern_filtering -sample_per_type 2` | 每种 scan pattern 类型抽样 2 个 |
| `write_patterns ...serial.v -verilog -serial ...` | 导出抽样串行测试平台 |

`SIM_KEEP_PATH 1` 保留导出命令给出的 pattern 数据文件路径，不会自动让所有相对路径变绝对路径。文件移动或改变仿真工作目录时，必须同时处理 vec/cfg 等配套文件引用。

历史外部模式有 9 个逻辑向量（5 basic + 4 clock_sequential）；过滤后串行导出为 4 个逻辑向量。原脚本没有显式导出 STIL，但保存了 TSDB 和 Verilog testbench。

## 9. 第六步：6_internal_atpg

这一阶段读取完整 scan 设计而非 graybox：

```tcl
import_scan_mode int_mode
set_current_mode int_mode_sa -type internal
...
set_system_mode analysis
create_patterns
write_tsdb_data -replace
```

`import_scan_mode` 在 `set_current_mode` 前执行，避免导入模式覆盖所需类型。内部 ATPG 得到 164 个逻辑向量（136 basic + 28 clock_sequential），对应 6 条 46 单元链。

同样先保存完整 TSDB、导出完整并行文件，再把串行导出抽成 4 个逻辑向量；内部模式日志还报告保存 6 个 chain test。抽样只作用于 scan/ATPG 部分，不能把整个文件总数简单算成 4。

最后执行：

```tcl
read_faults -mode ext_multi_stuck -fault_type stuck -merge
report_statistics -detail
```

这是把 external 模式故障检测分类合并到当前结果，用于计算核心整体覆盖率；**不会把 external 向量自动追加进现有 internal 向量集合**。历史日志还明确警告 previous internal test pattern set has been deleted，因此最后报告 `#test_patterns=0`。这不表示前面的 164 个向量从未生成，完整数据库和导出操作都发生在合并之前。

## 10. 覆盖率必须分模式、分列解释

| 模式/结果 | TC total | TC relevant | FC total | FC relevant |
|---|---:|---:|---:|---:|
| external | 26.85% | 83.66% | 21.70% | 48.08% |
| internal（合并前） | 86.39% | 97.42% | 82.40% | 92.38% |
| internal + external 故障合并 | 88.06% | 98.84% | 84.00% | 93.75% |

| 故障集合 | total | relevant |
|---|---:|---:|
| external graybox | 4,788 | 2,161 |
| internal 完整设计 | 16,820 | 15,003 |
| 合并后 | 16,820 | 15,071 |

external 与 internal 统计范围不同，不能只把两列百分比取平均。合并后 relevant 故障数量也发生变化，不能假设分母固定。`atpg_effectiveness` external=99.87%、internal=99.96%、合并后=99.93%，它不是 FC 的另一个名字。

## 11. 与顶层集成的接口关系

顶层脚本会 `open_tsdb` 打开本子目录数据库，然后用 `read_design add_ram_inv -design_id scan -view graybox` 读测试接口；顶层故障统计可以 `read_faults -module add_ram_inv -mode int_mode_sa ...` 合并内部测试覆盖信息。

retargeting 阶段会读取 int_mode_sa 的 patdb，而不是用 external 灰盒向量去替代内部模式。核心只有两条外部 EDT 输入/输出通道，但内部为 6 条链；wrapper 的 ext_mode 又是独立的 2 链配置，三种数字不能混为一谈。

## 12. 警告与验证边界

1. 库、SDC 和综合脚本中的 Linux 绝对路径仍需迁移；子目录的 `../../lib` 相对位置要保留。
2. RAM 旁路模式下的不可写/不可观察警告不能当作 MBIST 阵列通过证明。
3. graybox 的正常用途是上层边界测试及重定向，不能直接拿它运行核心完整内部 ATPG。
4. 配套 Verilog testbench 已生成，但本子目录没有其 gate-level simulation 通过 transcript；生成成功不等于仿真通过。
5. 增加 wrapper 和扫描单元后，应以最终链长统计为准；这里 276 不是原来 16 个功能 dff 的简单改名。

## 13. 准备依赖后的操作顺序

工作目录为 hier_flow/flow/3_add_ram_inv 的可写副本。按六个脚本顺序逐会话运行，每步保存新日志；前两步需要 DC 和 `.synopsys_dc.setup` 中匹配的 adk.db。

```bash
cd /实际材料路径/hier_flow/flow/3_add_ram_inv
mkdir -p patterns
tessent -shell -log 1_insert_mbist.new.log -replace
```

在 Tessent 中依次使用对应 `dofile`，含 `&` 的文件写成 `dofile {2_insert_edt&occ}`。各步通过后再进入下一步，最后完成 5_ext_atpg 和 6_internal_atpg，才把 TSDB 提供给顶层。

## 14. 建议的核对顺序

确认 3 个 RAM 及算法描述 → 确认两个 OCC、2 对 EDT 通道及内外控制信号 → 检查 wrapper 报告与 6/2 条模式链 → 检查 graybox 内容和 TCD → 分别核对 9 / 164 个逻辑向量 → 核对合并前后的故障口径 → 最后检查顶层 `add_ram_inv_1` 的实例映射。


## 附录：原始脚本对照

以下摘录原脚本，保留注释和原有拼写，便于逐行对照正文；它们不是本次修订后的可直接运行脚本。

### 1_insert_mbist

来源：[1_insert_mbist](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/1_insert_mbist)。

```tcl
set top add_ram_inv

set_context dft -no_rtl -design_id mbist
set_tsdb_output_directory ./tsdb

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory
read_verilog ./netlist/*.v

set_current_design $top

set_design_level physical_block

#add_dft_signals tck_occ_en

set_dft_specification_requirements -boundary_scan off -memory_test on -logic_test off

report_memory_instances

add_clocks 0 clk -period 5ns
add_clocks 0 ramclk -period 5ns

check_design_rules

set spec [create_dft_specification]
report_config_data $spec

process_dft_specification

extract_icl

run_synthesis
```

### 2_insert_edt&occ

来源：[2_insert_edt&occ](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/2_insert_edt&occ)。

```tcl
set top add_ram_inv

set_context dft -no_rtl -design_id edt

set_tsdb_output_directory ./tsdb

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory

read_design $top -design_id mbist -verbose

set_current_design $top

add_dft_signals ltest_en
add_dft_signals scan_en edt_update test_clock -source_node { scan_en edt_update test_clock }
add_dft_signals shift_capture_clock edt_clock -create_from_other_signals
add_dft_signals memory_bypass_en
add_dft_signals tck_occ_en

##if flat flow there use edt_mode only
add_dft_signals int_mode ext_mode  
add_dft_signals int_ltest_en ext_ltest_en

report_dft_signal

set_dft_specification_requirements -logic_test on
check_design_rules

set spec [create_dft_specification -sri_sib_list {edt occ} ]
report_config_data $spec

read_config_data -in $spec -from_string {
  OCC {
    ijtag_host_interface : Sib(occ);
  }
}

set id_clk_list [list \
  clk clk \
  ramclk ramclk \
]
foreach {id clk} $id_clk_list {
  set occ [add_config_element OCC/Controller($id) -in $spec]
  set_config_value clock_intercept_node -in $occ $clk
}
report_config_data $spec


read_config_data -in $spec -from_string {
   EDT {
     ijtag_host_interface : Sib(edt);
       Controller (c1) {
         longest_chain_range : 2, 100;
         scan_chain_count : 6;
         input_channel_count : 2;
         output_channel_count : 2;
       }
    }
}

report_config_data $spec

process_dft_specification
extract_icl

run_synthesis
```

### 3_insert_scan

来源：[3_insert_scan](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/3_insert_scan)。

```tcl
set top add_ram_inv

set_context dft -no_rtl -scan -design_id scan

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory

set_tsdb_output_directory ./tsdb

read_design $top -design_id edt -verbose

set_current_design $top

set_system_mode analysis
report_scan_segments
report_clocks

set_wrapper_analysis_options -exclude_ports [ get_ports {*edt_channels*} ]
analyze_wrapper_cells
report_wrapper_cells -verbose

set edt_instance [get_instance -of_icl_instances [get_icl_instances -filter tessent_instrument_type==mentor::edt]]
add_scan_mode int_mode -edt_instance $edt_instance
add_scan_mode ext_mode -chain_count 2

analyze_scan_chains
report_scan_chains

insert_test_logic

report_scan_chains

#write_design -output_file ../scan/${top}_scan.v -replace
#write_atpg_setup ../scan/setup -replace
```

### 4_gen_graybox

来源：[4_gen_graybox](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/4_gen_graybox)。

```tcl
set top add_ram_inv

set_context patterns -scan -design_id scan

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory

set_tsdb_output_directory ./tsdb

# Read in the scan inserted netlist/design
read_design $top -design_id scan -verbose
set_current_design $top

report_dft_signals

# DRC check the scan chains in each mode and create graybox for external mode
foreach_in_collection mode_wrapper [get_config_elements Core($top)/Scan/Mode -part tcd -silent] {
  set mode_name [get_config_value $mode_wrapper -id <0>]
  set mode_type [get_config_value type -in $mode_wrapper]
  import_scan_mode $mode_name
  check_design_rules
  if {$mode_type eq "external"} {
    report_scan_cells
    analyze_graybox
    write_design -tsdb -graybox -verbose
    write_core_description ./${top}_gray_box.tcd -replace
  }
  set_system_mode setup

}
```

### 5_ext_atpg

来源：[5_ext_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/5_ext_atpg)。

```tcl
set top add_ram_inv

set_context patterns -scan

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory

set_tsdb_output_directory ./tsdb

# Read the design
read_design $top -design_id scan -view graybox -verbose
set_current_design $top

# Specify the current mode using a different name than what was used during scan insertion with add_scan_mode command
set_current_mode ext_multi_stuck -type external

report_dft_signals

# Extract the external mode specified during scan insertion to run ATPG
import_scan_mode ext_mode
report_core_instances
report_static_dft_signal_settings

report_clocks
add_black_boxes -auto
set_system_mode analysis
report_clocks

set_xclock_handling x

# Generate patterns
create_patterns

# Store TCD, flat_model, fault list and patDB format files in the TSDB directory
write_tsdb_data -replace

# Write Verilog patterns for simulation
write_patterns ./patterns/${top}_ext_stuck_parallel.v -verilog -parallel -replace -parameter_list {SIM_KEEP_PATH 1}
set_pattern_filtering -sample_per_type 2
write_patterns ./patterns/${top}_ext_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}
```

### 6_internal_atpg

来源：[6_internal_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/3_add_ram_inv/6_internal_atpg)。

```tcl
set top add_ram_inv

set_context patterns -scan

read_cell_library ../../lib/adk.tcelllib
read_cell_library ../../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../../lib -extension tcd_memory

set_tsdb_output_directory ./tsdb

# Read the design
read_design $top -design_id scan -verbose
set_current_design $top

import_scan_mode int_mode
set_current_mode int_mode_sa -type internal

report_core_instances
report_static_dft_signal_settings

report_clocks

set_system_mode analysis
report_clocks

# Generate patterns
create_patterns

# Store TCD, flat_model, fault list and patDB format files in the TSDB directory
write_tsdb_data -replace

# Write Verilog patterns for simulation
write_patterns ./patterns/${top}_int_stuck_parallel.v -verilog -parallel -replace -parameter_list {SIM_KEEP_PATH 1}
set_pattern_filtering -sample_per_type 2
write_patterns ./patterns/${top}_int_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}

read_faults -mode ext_multi_stuck -fault_type stuck -merge
report_statistics -detail
```
