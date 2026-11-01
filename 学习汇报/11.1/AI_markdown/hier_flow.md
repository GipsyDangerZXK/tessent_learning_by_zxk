# hier_flow 实验流程与命令详解

实验目录：[hier_flow](E:/tessent/Reference_2026/tessent/hier_flow)；顶层工作目录为 `hier_flow/flow`。

本说明按“实验目的 → 文件与输入输出 → 执行顺序 → 逐命令解释 → 结果解读 → 差异和问题 → 复现检查”的结构编写。脚本代表配置意图，日志或产物代表历史证据；没有证据的结果不作推定。本次仅阅读和核对文件，没有重新执行 Tessent、Design Compiler 或仿真器，也没有修改 E 盘的实验源文件。

文中的 Bash 命令用于安装了 EDA 工具的 Linux 环境；E 盘是当前 Windows 上的材料位置。`dofile` 后的脚本在 Tessent 中执行，`.proc` 是测试过程描述，不是普通 Tcl。迁移路径必须按原有目录相对关系处理。

历史工具为 Tessent 2021.3。这个包同时保留不同日期的网表、配置与日志，有明确的错误记录和版本差异；理解这些差异是复现的一部分，不能只按 1～9 的编号认定已经从头无误跑通。

## 1. 这个实验要完成什么

层次化 DFT 的重点是：先让核心独立完成内部测试，再在顶层保留测试访问和边界逻辑，通过 retargeting 把核心已有向量转换成芯片端口可施加的向量。

功能结构由原始 top.v 确认：

```text
net_in_1[15:0] → add_ram_1 ─┐
                           ├→ add_ram_inv_1 → out/net_out[7:0]
net_in_2[15:0] → add_ram_2 ─┘

每个核心内部：3 个 RAM + 2 个 adder + 16 个功能 dff
```

2 个 add_ram 实例复用同一种核心设计；add_ram_inv 是独立核心类型，功能寄存器用 QB 输出。因此不能用 add_ram 的向量代替 add_ram_inv 的向量。

测试流程：

```text
1_insert_pad（芯片 I/O 准备）
      ├─ 2_add_ram：六步核心 DFT → graybox + int_mode_sa 向量/故障
      └─ 3_add_ram_inv：六步核心 DFT → graybox + int_mode_sa 向量/故障
                        ↓
4_insert_bscan&mbist：顶层 TAP/BSCAN，打开两个核心 TSDB
5_insert_edt_occ：顶层 EDT/OCC + 模式化 TAM 连接
6_insert_scan：只对顶层/边界相关逻辑分配扫描链
      ├─ 7_atpg：顶层模式 ATPG + 核心故障信息合并
      ├─ 8_retargeting1：复用 add_ram 向量同时测试两个实例
      └─ 9_retargeting2：复用 add_ram_inv 向量测试一个实例
```

核心独立步骤分别详见 [2_add_ram.md](C:/Users/zxk__/Desktop/GitHub/markdown/2_add_ram.md)、[3_add_ram_inv.md](C:/Users/zxk__/Desktop/GitHub/markdown/3_add_ram_inv.md)。本文详细说明顶层集成，避免把灰盒模型中的省略逻辑当作真实被删除的功能电路。

## 2. 文件与阶段依赖

| 文件 | 阅读用途 |
|---|---|
| [1_insert_pad](E:/tessent/Reference_2026/tessent/hier_flow/flow/1_insert_pad) | 顶层 PAD 插入 |
| [4_insert_bscan&mbist](E:/tessent/Reference_2026/tessent/hier_flow/flow/4_insert_bscan&mbist) | 顶层 BoundaryScan/TAP 集成 |
| [5_insert_edt_occ](E:/tessent/Reference_2026/tessent/hier_flow/flow/5_insert_edt_occ) | EDT/OCC、模式控制和 TAM 连接 |
| [6_insert_scan](E:/tessent/Reference_2026/tessent/hier_flow/flow/6_insert_scan) | 顶层扫描插入 |
| [7_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/7_atpg) | 顶层 ATPG 及核心覆盖率合并 |
| [8_retargeting1](E:/tessent/Reference_2026/tessent/hier_flow/flow/8_retargeting1) | add_ram 双实例重定向 |
| [9_retargeting2](E:/tessent/Reference_2026/tessent/hier_flow/flow/9_retargeting2) | add_ram_inv 重定向 |
| [top.v](E:/tessent/Reference_2026/tessent/hier_flow/netlist/top.v) | 原始功能层次 |
| [top.v](E:/tessent/Reference_2026/tessent/hier_flow/flow/netlist/top.v) | 历史 PAD 后网表，输出已名为 net_out |

| 数据 | 当前状态 |
|---|---|
| hier_flow/lib | adk/picdram 的 DFT、综合、存储器描述和仿真库均存在 |
| 两个核心 TSDB | 存在，各包含 mbist/edt/scan、graybox 和内外模式数据库 |
| 顶层 TSDB | 存在 bscan/edt/scan 和 ATPG/retargeting 模式 |
| 顶层 patterns | 有 top_stuck 的串/并行文件和 STIL，以及一个 add_ram_stuck.stil |
| 门级仿真通过记录 | 没有找到最终 ATPG/retargeting 文件对应的仿真 transcript |

### 2.1 最先要处理的源版本差异

原始 `hier_flow/netlist/top.v` 输出名为 `out`，1_insert_pad.log 也遍历了 `out[7:0]`；但当前 `flow/netlist/top.v` 输出已经是 `net_out`，第 4、5 步辅助 I/O 与 TAM 也使用 net_out。

第 1 步脚本没有重命名 out→net_out 的命令，所以现有功能源、PAD 后网表和后续脚本不是一个完全一致的重建链。重跑前应统一输出命名，或明确采用经过核对的 PAD 后网表作为起点；本次没有改写它们。

## 3. 三个测试层级及模式名

| 层级/模式 | 负责什么 |
|---|---|
| 核心 int_mode | 6 条 46 单元内部链，由核心 EDT 驱动，核心逻辑内部 ATPG |
| 核心 ext_mode | 2 条 wrapper 链，供边界/互连测试 |
| 顶层 edt_mode | 1 条顶层扫描链，由顶层 EDT 访问 |
| retargeting1_mode | 选择 add_ram_1 和 add_ram_2 的通道 |
| retargeting2_mode | 选择 add_ram_inv_1 的通道 |
| int_mode_sa / edt_top_stuck / retarget1_stuck / retarget2_stuck | 数据库中的 ATPG/重定向结果模式名 |

`design_id` 的 mbist、bscan、edt、scan 用于设计阶段；上述 mode 用于某阶段中的测试配置/结果。它们不是互相替换的参数。

## 4. 第一步和两个子实验

### 4.1 顶层 PAD 插入

`set_context dft -no_rtl`、`read_cell_library`、`read_verilog ../netlist/*.v` 读取门级层次；`set_current_design top` 后进入 insertion。

原有输入连接通过 `move_connection` 移到新 ipad/C，再将端口接 ipad/PAD；输出连接移到 opad/I，端口接 opad/PAD。`foreach_in_collection`、`get_ports`、`get_name_list`、`lindex` 和 `string map` 用于遍历/命名。

新增 TDI/TMS/TRST/TCK/scan_en/edt_update/test_clock 和 TDO，最终 `write_design -output_file ./netlist/top.v`。该行未带 -replace，历史日志却有 -replace，也是脚本与日志的一个小差异。

`create_instance` 负责建立 ipad/opad 实例，`create_port` 创建新增顶层接口，`create_connection` 建立端口、PAD 和内部网络的连接；它们分别改变设计对象、接口和连接，不能只创建 PAD 实例就认为端口路径完整。

### 4.2 先生成核心可交付数据

两个子实验各执行 MBIST → EDT/OCC → wrapper/scan → graybox → external ATPG → internal ATPG。顶层后续依赖它们的 graybox、TCD/ICL、int_mode_sa 的 patdb 和 faults；仅有原始 add_ram.v 不足以做 retargeting。

两核心的统一结构参数：2 个 OCC，2 输入/2 输出 EDT 通道，6 条内部链共 276 单元，2 条外部 wrapper 链共 41 单元。add_ram 内部生成 162 个逻辑向量，add_ram_inv 为 164 个；细节和覆盖率在各自文档中。

## 5. 第四步：4_insert_bscan&mbist

文件名包含 mbist，但顶层有效需求只写了 `-boundary_scan on`；MBIST 已在核心内插入，不应按文件名宣称这里又为所有 RAM 创建一套顶层 MBIST。

| 命令 | 作用 |
|---|---|
| `set_tsdb_output_directory ./tsdb` | 设置顶层数据库的输出位置 |
| `open_tsdb ./2_add_ram/tsdb` | 打开核心数据库供引用，保持该库独立 |
| `open_tsdb ./3_add_ram_inv/tsdb` | 打开另一种核心数据库 |
| `set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory` | 配置 SRAM 测试描述的搜索路径和扩展名 |
| `read_verilog ./netlist/top.v` | 读取芯片 PAD 后网表 |
| `set_design_level chip` | 按芯片顶层生成 TAP/BSCAN |
| `set_attribute_value ... -name function ...` | 将五个 JTAG 端口关联到 TAP 功能 |
| `set_boundary_scan_port_options clk/ramclk -cell_options clock` | 按时钟端口处理 |
| 测试端口的 `dont_touch` | 保持 scan_en/edt_update/test_clock 的测试访问路径 |
| `add_dft_signals scan_en -source_nodes scan_en` | 声明扫描使能来源 |
| `add_clocks 0 clk/ramclk -period 5ns` | 声明两个功能时钟及关闭状态 |
| `check_design_rules` | 核对结构和边界扫描条件 |
| `create_dft_specification` / `report_config_data` | 建立并检查配置 |

辅助 I/O 配置为：输入 `net_in_1[7]、[0]、[1]`；输出 `net_out[7]、[0]、[1]、[2]、[3]`。它们分别用于顶层 EDT 和两种核心重定向路径。

这一设置由 `read_config_data -in ${spec}/BoundaryScan -from_string` 写入；`auxiliary_input_ports` 和 `auxiliary_output_ports` 是配置属性，不是新建 Verilog 端口的命令。

`process_dft_specification → extract_icl → run_synthesis` 生成/插入顶层测试访问结构、提取网络并综合新增仪器。历史综合为 pass 1 fail 0。

## 6. 第五步：5_insert_edt_occ

### 6.1 读入顶层全设计和核心灰盒

```tcl
read_design top -design_id bscan -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose
```

`-view graybox` 只把核心上层测试所需部分带入当前分析，避免每次重复展开完整内部逻辑；实际功能实例依然属于芯片设计。

### 6.2 声明芯片控制信号

| 信号/命令组 | 含义 |
|---|---|
| `add_dft_signals int_ltest_en output_pad_disable` | 支持通过边界扫描控制输入/输出 PAD 路径的逻辑测试设置 |
| tck_occ_en、ltest_en | scan-tested 仪器时钟控制和逻辑测试使能 |
| edt_mode | 顶层 EDT 模式 |
| test_clock、edt_update 的 source_nodes | 使用已有顶层测试端口 |
| shift_capture_clock、edt_clock 的 create_from_other_signals | 创建内部测试时钟 |
| retargeting1_mode、retargeting2_mode | 定义核心测试访问的两种选择模式 |

原脚本只是声明 int_ltest_en/output_pad_disable，7_atpg 没有显式把它们设为 1；不能因为名字出现就说当前模式已经采用“不接触功能 PAD”的测试方式。

### 6.3 add_dft_modal_connections：建立随模式切换的 TAM

TAM 是 Test Access Mechanism。这里把芯片引脚与不同 EDT 仪器端口按模式连接。

| 选项 | 含义 |
|---|---|
| `-ports` | 芯片可复用的外部端口 |
| `-input_data_destination_nodes` | 输入数据送到的核心测试节点 |
| `-output_data_source_nodes` | 输出数据来自哪个测试节点 |
| `-enable_dft_signal` | 哪个 DFT 模式使能这一连接 |

顶层 EDT 的两条声明用 `*` 作为 destination/source 占位，脚本注释明确表示此处不建立具体数据连接，稍后通过 EDT Connections 指定。不要把它当成连接所有设计节点的通配命令。

| 模式 | 输入 | 输出 | 被测对象 |
|---|---|---|---|
| edt_mode | net_in_1[7] | net_out[7] | 顶层 EDT |
| retargeting1_mode | net_in_1[0:1]，同时广播给两个 add_ram | net_out[0:1] 对 add_ram_1，net_out[2:3] 对 add_ram_2 | 2 个相同核心实例 |
| retargeting2_mode | net_in_1[0:1] | net_out[0:1] | add_ram_inv_1 |

相同核心可以共享输入比特流，但响应输出分别观察。两个重定向模式复用同一组部分 I/O，必须由模式控制分开使用。`report_dft_modal_connections` 用于核对最终映射。

原脚本残留 processor_core/gps_baseband 注释来自通用模板，真实模块名应以 add_ram/add_ram_inv 连接命令为准。

### 6.4 顶层 OCC/EDT 配置

`set_dft_specification_requirements -logic_test On` 后创建 `-sri_sib_list {occ edt}`，分别为 clk 和 ramclk 增加 OCC Controller，并指定 clock_intercept_node。

| 顶层 EDT 参数 | 值 |
|---|---|
| longest_chain_range | 2～100 |
| scan_chain_count | 1 |
| input_channel_count / output_channel_count | 1 / 1 |
| 连接 | 辅助 net_in_1[7] → EDT → 辅助 net_out[7] |

`set_config_value port_pin_name` 配合 `get_auxiliary_pins` 和 `get_single_name` 设置数据接口；`process_dft_specification`、`extract_icl`、`run_synthesis` 是后续实际生成/映射步骤。

`ijtag_host_interface : Sib(occ)/Sib(edt)` 把相应仪器关联到 IJTAG 的选择分支，是 DFT 配置属性；访问控制通过生成的 SIB 网络实现。

### 6.5 该阶段日志存在真实错误

5_insert_edt_occ.log 的 OCC 循环被截断，出现 `missing close-bracket`；随后处理配置又报 `OCC: Controller wrapper is not specified`。当前脚本中对应循环是完整写法，说明日志不是当前脚本一次干净运行的证明。

现有 TSDB 的 top_edt.dft_spec 有两个 OCC Controller；top_edt/top.vg 的文件头时间为 2022-05-13，内含两个 OCC 实例，而该主日志是 6 月运行记录。后续 top_scan 网表时间为 2022-06-27。这里混有旧产物/不同会话结果，不能用最后 `pass 1 fail 0` 抹掉前面的配置错误。

## 7. 第六步：6_insert_scan

| 命令 | 作用 |
|---|---|
| `open_tsdb` 两次 | 继续提供核心数据库 |
| `read_design top -design_id edt` | 读取顶层 EDT/OCC 设计 |
| 两次核心 `-view graybox` | 保持上层可见结构 |
| `set_system_mode analysis` | 执行规则和扫描段分析 |
| `report_scan_segments`、`report_clocks` | 核对已有 wrapper/仪器扫描段及测试时钟 |
| `get_icl_module -of_instances top* -filter ...mentor::edt` | 仅筛选顶层相关 EDT，避免误选子核心 EDT |
| `get_name/get_instance/get_name_list` | 把 ICL 模块映射为设计实例名 |
| `add_scan_mode edt_mode -edt_instance ...` | 创建顶层扫描配置 |
| `analyze_scan_chains`、`report_scan_chains` | 规划并报告链 |
| `insert_test_logic` | 实际实现，并写顶层 scan 阶段数据 |

历史顶层新增 1 条长度 **129** 的链，新增实例 3、顶层端口 0。报告还列出子核心 int_mode 链 length=0；那是当前灰盒视图未展开核心内部单元，不能说 18 条核心链物理上都是空链。核心独立流程及 retargeting 报告保留了真实 276 单元/核心的信息。

## 8. 第七步：7_atpg

### 8.1 顶层模式生成

读取顶层 scan 和两种 graybox，设置 `edt_top_stuck`，导入 edt_mode，报告核心实例；显式设置 `tck_occ_en=1`，进入 analysis 并 `create_patterns`。

`set_current_mode` 选择 ATPG 结果模式，`import_scan_mode edt_mode` 恢复第 6 步扫描配置；`report_core_instances` 核对核心实例，`report_static_dft_signal_settings` 与 `report_clocks` 检查最终静态值和时钟，而不是产生额外测试结构。

历史分析追踪到顶层一条 129 单元链，并警告：

- 最长链 129 超过 EDT 生成时的 2～100 范围；
- compression per pattern=0.94，不大于 1；
- 为初始化 non-scan 元素自动增加 post-shift，且有 shadow 元素影响并行 testbench。

因此配置中有 EDT，并不代表这个顶层模式实现了有效的数据压缩。重跑应根据真实链长评估 EDT 参数，而非只复制旧范围。

### 8.2 保存和导出顺序

```tcl
write_tsdb_data -replace
write_patterns ./patterns/${top}_stuck_parallel.v -verilog -parallel -replace -scan -parameter_list {SIM_KEEP_PATH 1}
set_pattern_filtering -sample_per_type 2
write_patterns ./patterns/${top}_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}
write_patterns ./patterns/${top}_stuck.stil -stil -replace
```

重要区别：**STIL 位于抽样命令之后。** 日志确认 8 个逻辑向量中仅选 4 个导出串行文件和 STIL；另包含 4 个 chain test。若需要完整 STIL，应在导出前执行 `set_pattern_filtering -off`，或把完整 STIL 导出放到抽样前。

当前 top_stuck.stil 的 Header pattern_end=3，正文只有 pattern 0～3，证实保存的是抽样逻辑集合。Header 仍有完整 ATPG coverage 注释，不能把它当作“这 4 个向量单独重新测得了完整 coverage”。

### 8.3 合并核心内部故障覆盖信息

```tcl
read_faults -module add_ram -mode int_mode_sa -fault_type stuck -merge -graybox
read_faults -module add_ram_inv -mode int_mode_sa -fault_type stuck -merge -graybox
report_statistics -detail
```

第一条将 add_ram 核心结果映射到两个对应实例，第二条合并 add_ram_inv。`-graybox` 允许结合灰盒上层模型处理核心已知故障覆盖信息。

日志明确提示旧 internal pattern set 被删除；合并后的 #test_patterns=0 是当前向量集合状态，不等于前面没生成向量，也不代表自动生成了一个覆盖全部核心的新向量文件。`write_tsdb_data` 在合并前执行，所以也不能假定此前 patdb 已经包含合并后的向量或统计。

## 9. 顶层覆盖率：合并前后与四列统计

### 9.1 顶层模式单独 ATPG

| 指标 | total | total relevant |
|---|---:|---:|
| 故障数 | 20,822 | 10,144 |
| Test coverage | 32.84% | 83.98% |
| Fault coverage | 27.66% | 56.78% |
| ATPG effectiveness | 99.89% | 99.89% |

产生 8 个逻辑向量（3 basic + 5 clock_sequential），生成过程中模拟过 18 个。导出时另有 chain test，且抽样行为见上一节。

### 9.2 核心结果合并后

最终报告多了一层 with/w/o unlisted faults 分类，必须保留完整表头：

| 指标 | with unlisted total | with unlisted relevant | w/o unlisted total | w/o unlisted relevant |
|---|---:|---:|---:|---:|
| 故障数 | 57,278 | 49,241 | 20,822 | 12,785 |
| Test coverage | 83.34% | 97.74% | 54.97% | 93.59% |
| Fault coverage | 79.33% | 92.28% | 51.41% | 83.73% |
| ATPG effectiveness | 99.93% | 99.93% | 99.81% | 99.81% |

with unlisted 把核心内部省略在当前灰盒明细之外、但有可继承覆盖信息的故障计入综合统计。不要从这四列中挑一个最高值，去与合并前另一种范围比较并声称算法提高了相应百分比。

## 10. 第八步：8_retargeting1

### 10.1 选择核心和模式

```tcl
set_context patterns -scan_retargeting
set_current_mode retarget1_stuck
set_static_dft_signal_values retargeting1_mode 1
add_core_instances -instances {add_ram_1 add_ram_2} -core add_ram -mode int_mode_sa
```

| 命令 | 作用 |
|---|---|
| `patterns -scan_retargeting` | 复用并转换核心向量，不是在芯片层重新搜索一遍核心 ATPG |
| `set_static_dft_signal_values` | 选择第 5 步已建立的 TAM 模式 |
| `add_core_instances -instances ... -core ... -mode ...` | 将两个物理实例关联到 add_ram 核心 int_mode_sa 描述/向量 |
| `report_core_descriptions/report_clocks` | 检查关联核心和时钟 |
| `set_system_mode analysis` | 建立重定向分析与过程模型 |
| `write_tsdb_data` | 此处在 read_patterns 前保存重定向模式描述，不等于已经保存输出向量 |
| `read_patterns -module add_ram -fault_type stuck -mode int_mode_sa` | 从核心 TSDB 读取已有 patdb |
| `set_external_capture_options -pll_cycles 5 [lindex [get_timeplate_list] 0]` | 为导出的芯片级向量指定相应时序模板和固定 PLL 捕获周期设置；不会自动把 stuck-at 改成 transition |
| `write_patterns ... -stil -replace` | 经访问网络和时钟过程转换，写出顶层可施加 STIL |

历史报告：2 个 core instances，shift cycles=55，scan channels=4，scan chains=12，scan cells=552。这里 552=2×276、12=2×6；mode 1 输入广播仍只复用两个芯片输入位，报告的 4 个核心通道不能解读成 4 个独立物理输入引脚。

## 11. 第九步：9_retargeting2

同样使用 scan_retargeting，但选择 retargeting2_mode，并执行：

```tcl
add_core_instances -instances {add_ram_inv_1} -core add_ram_inv -mode int_mode_sa
read_patterns -module add_ram_inv -fault_type stuck -mode int_mode_sa
```

历史报告为 1 个核心、55 shift cycles、2 channels、6 chains、276 cells；来源是 add_ram_inv 的 164 个内部逻辑向量。

### 11.1 两个重定向输出会覆盖

8 和 9 两个脚本最后都写：

```tcl
write_patterns ./patterns/add_ram_stuck.stil -stil -replace
```

因此依次执行会让第 9 步覆盖第 8 步。当前实际保存的该文件 Header 为 pattern_end=163，含 add_ram_inv_1 的注释而不含 add_ram_1，支持它是第 9 步输出的判断。

建议在实验副本中把第 9 步输出改成 `add_ram_inv_stuck.stil`，或统一使用包含 retarget 模式名的文件名。本文只说明问题，没有改写原脚本或现有 STIL。

## 12. 目前可以确认与不能确认的结果

| 事项 | 证据与结论 |
|---|---|
| 核心 DFT/graybox/ATPG | 两套独立脚本和日志有结构、链数、模式数据库证据 |
| 顶层 BSCAN 综合 | 有 pass 1 fail 0 |
| 当前 5_insert_edt_occ 原样一次通过 | 不能确认，日志有 bracket/OCC 配置错误且混用旧产物 |
| 顶层扫描和 ATPG | 有 129 单元链、8 个逻辑向量及统计 |
| 两种 retargeting | 日志有核心映射、patdb 读取和 STIL 写出记录 |
| 两份 retargeting STIL 都留存 | 否，输出同名造成覆盖，当前只见一个文件 |
| 最终向量门级仿真通过 | 未找到对应 transcript，不能宣称通过 |

## 13. 准备环境后的操作顺序

先在可写副本中统一 out/net_out，迁移 `.synopsys_dc.setup` 路径，处理第 9 步同名输出，并保留历史 TSDB 作对照。若要验证完整可重建性，应在新的输出目录完成每阶段，避免旧文件掩盖新命令失败；不要清空原归档后才开始检查。

| 次序 | 工作目录 | 入口 |
|---|---|---|
| 1 | 顶层 flow | 1_insert_pad |
| 2 | flow/2_add_ram | 本核心 1～6 |
| 3 | flow/3_add_ram_inv | 本核心 1～6 |
| 4～7 | 回到顶层 flow | 4、5、6、7 逐步执行 |
| 8 | 顶层 flow | 8_retargeting1，立即确认/保留独立输出 |
| 9 | 顶层 flow | 9_retargeting2，使用不同输出文件 |

```bash
cd /实际材料路径/hier_flow/flow
mkdir -p netlist patterns
tessent -shell -log 4_insert_bscan.new.log -replace
```

Tessent 中 `dofile {4_insert_bscan&mbist}`；每阶段检查后退出，再开新会话。子核心运行时需切换到对应子目录，才能正确解释 `../../lib` 和 `./tsdb`。

## 14. 最值得追踪的五条关系

1. 顶层功能实例与两个核心类型是否正确映射。
2. 核心 int/ext 模式、graybox 和上层 EDT 的测试边界是否清楚。
3. 各 retargeting mode 的输入广播和输出选择是否按第 5 步连接。
4. 覆盖率继承/合并与真实向量生成/转换是否区分。
5. 文件版本、抽样顺序和输出文件名是否使最终结果完整可追溯。


## 附录：原始脚本对照

以下摘录原脚本，保留注释和原有拼写，便于逐行对照正文；它们不是本次修订后的可直接运行脚本。

### 1_insert_pad

来源：[1_insert_pad](E:/tessent/Reference_2026/tessent/hier_flow/flow/1_insert_pad)。

```tcl
set top top

set_context dft -no_rtl

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_verilog ../netlist/*.v

set_current_design $top

set_system_mode insertion

set inputPortList [ get_ports -of_module $top -direction input ]
foreach_in_collection inputPort $inputPortList { 
    set portName [lindex [get_name_list $inputPort] 0]
    set padName  [ string map { \[ _ \] _ } ${portName}_PAD ]
    create_instance $padName -of_module ipad
    move_connection -from $inputPort -to $padName/C
    create_connection $inputPort $padName/PAD
}

set outputPortList [ get_ports -of_module $top -direction output ]
foreach_in_collection outputPort $outputPortList { 
    set portName [lindex [get_name_list $outputPort] 0]
    set padName  [ string map { \[ _ \] _ } ${portName}_PAD ]
    create_instance $padName -of_module opad
    move_connection -from $outputPort -to $padName/I
    create_connection $outputPort $padName/PAD
}


set inputPortList { TDI TMS TRST TCK scan_en edt_update test_clock}
set outputPortList { TDO }

foreach portName $inputPortList {
    set padName ${portName}_PAD
    create_instance $padName -of_module ipad
    create_port $portName -direction input
    create_connection $portName $padName/PAD
}

foreach portName $outputPortList {
    set padName ${portName}_PAD
    create_instance $padName -of_module opad
    create_port $portName -direction output
    create_connection $portName $padName/PAD
}

write_design -output_file ./netlist/${top}.v
```

### 4_insert_bscan&mbist

来源：[4_insert_bscan&mbist](E:/tessent/Reference_2026/tessent/hier_flow/flow/4_insert_bscan&mbist)。

```tcl
set top top

set_context dft -no_rtl -design_id bscan
set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_verilog ./netlist/${top}.v

set_current_design $top

set_design_level chip

set_dft_specification_requirements -boundary_scan on 

set_attribute_value TCK -name function -value tck
set_attribute_value TDI -name function -value tdi
set_attribute_value TMS -name function -value tms
set_attribute_value TRST -name function -value trst
set_attribute_value TDO -name function -value tdo

set_boundary_scan_port_options clk -cell_options clock
set_boundary_scan_port_options ramclk -cell_options clock
set_boundary_scan_port_options scan_en -cell_options dont_touch
set_boundary_scan_port_options edt_update -cell_options dont_touch
set_boundary_scan_port_options test_clock -cell_options dont_touch

add_dft_signals scan_en -source_nodes scan_en
add_clocks 0 clk -period 5ns
add_clocks 0 ramclk -period 5ns

check_design_rules

set spec [create_dft_specification]
report_config_data $spec
read_config_data -in ${spec}/BoundaryScan -from_string {
    AuxiliaryInputOutputPorts{
        auxiliary_input_ports : net_in_1[7], net_in_1[0], net_in_1[1] ;
        auxiliary_output_ports : net_out[7], net_out[0], net_out[1], net_out[2], net_out[3] ;
    }
}
report_config_data $spec

process_dft_specification

extract_icl

run_synthesis
```

### 5_insert_edt_occ

来源：[5_insert_edt_occ](E:/tessent/Reference_2026/tessent/hier_flow/flow/5_insert_edt_occ)。

```tcl
set top top

set_context dft -no_rtl -design_id edt
set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory

read_design top -design_id bscan -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose

set_current_design $top

# The design level is already specified in first pass and no need to specify it again

# Add DFT Signals
# DFT Signal used for BoundaryScan to be used with logic test without contacting inputs
add_dft_signals int_ltest_en output_pad_disable 
# DFT Signals used for Scan Tested Network with Instruments like memorybist/boundary scan
add_dft_signals tck_occ_en 
# DFT Signal used for logic test
add_dft_signals ltest_en 
# DFT Signal used by top-level EDT
add_dft_signals edt_mode 
add_dft_signals test_clock edt_update -source_nodes {test_clock edt_update}
# Create shift_capture_clock and edt_clock as gated versions of test_clock
add_dft_signals shift_capture_clock edt_clock -create_from_other_signals
# DFT Signals used for retargeting lower level ATPG patterns
add_dft_signals retargeting1_mode retargeting2_mode

# Simple TAM(Test Access Mechanism) connections
# For chip_top, edt_mode - The asterisk(*) below means do not make connection to the data inputs.
add_dft_modal_connections -ports net_in_1[7] -input_data_destination_nodes * -enable_dft_signal  edt_mode
add_dft_modal_connections -ports net_out[7] -output_data_source_nodes     * -enable_dft_signal edt_mode

# For processor_core - use retargeting_mode1
add_dft_modal_connections -ports net_in_1[0] -input_data_destination_nodes add_ram_inv_1/add_ram_inv_edt_controller_c1_edt_channels_in[0] -enable_dft_signal  retargeting2_mode
add_dft_modal_connections -ports net_in_1[1] -input_data_destination_nodes add_ram_inv_1/add_ram_inv_edt_controller_c1_edt_channels_in[1] -enable_dft_signal  retargeting2_mode
add_dft_modal_connections -ports net_out[0] -output_data_source_nodes add_ram_inv_1/add_ram_inv_edt_controller_c1_edt_channels_out[0] -enable_dft_signal  retargeting2_mode
add_dft_modal_connections -ports net_out[1] -output_data_source_nodes add_ram_inv_1/add_ram_inv_edt_controller_c1_edt_channels_out[1] -enable_dft_signal  retargeting2_mode

# For gps_baseband - use retargeting_mode2
add_dft_modal_connections -ports net_in_1[0] -input_data_destination_nodes add_ram_1/add_ram_edt_controller_c1_edt_channels_in[0] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_in_1[0] -input_data_destination_nodes add_ram_2/add_ram_edt_controller_c1_edt_channels_in[0] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_in_1[1] -input_data_destination_nodes add_ram_1/add_ram_edt_controller_c1_edt_channels_in[1] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_in_1[1] -input_data_destination_nodes add_ram_2/add_ram_edt_controller_c1_edt_channels_in[1] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_out[0] -output_data_source_nodes add_ram_1/add_ram_edt_controller_c1_edt_channels_out[0] -enable_dft_signal  retargeting1_mode 
add_dft_modal_connections -ports net_out[1] -output_data_source_nodes add_ram_1/add_ram_edt_controller_c1_edt_channels_out[1] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_out[2] -output_data_source_nodes add_ram_2/add_ram_edt_controller_c1_edt_channels_out[0] -enable_dft_signal  retargeting1_mode
add_dft_modal_connections -ports net_out[3] -output_data_source_nodes add_ram_2/add_ram_edt_controller_c1_edt_channels_out[1] -enable_dft_signal  retargeting1_mode

report_dft_modal_connections

# Specify pre-DFT DRC rules
set_dft_specification_requirements -logic_test On

check_design_rules
report_dft_control_points

# Create and report a DFT Specification
set spec [create_dft_specification -sri_sib_list {occ edt} ]
report_config_data $spec
read_config_data -in $spec -from_string {
  OCC {
    ijtag_host_interface : Sib(occ);
  }
}
# Below is a generic way to populate the OCC. The clock list is design-specific and needs to be updated for the design
# The scan_enable and shift_capture_clock signals are automatically connected to OCC instances
# Modify the below list for your specific design requirements
set id_clk_list [list \
     clk clk \
     ramclk ramclk \
]
foreach {id clk} $id_clk_list {
  set occ [add_config_element OCC/Controller($id) -in $spec]
  set_config_value clock_intercept_node -in $occ $clk
}
report_config_data $spec

# The edt_clock and edt_update signals are automatically connected to EDT instances
# The EDT controller is built with Bypass
# Modify the below specification for your specific design requirements
# The propery connect_bscan_segments_to_lsb_chains defaults to auto and connects the divided boundary scan segments from the previous pass to EDT
read_config_data -in $spec -from_string {
   EDT {
     ijtag_host_interface : Sib(edt);
       Controller (c1) {
        longest_chain_range : 2, 100;
        scan_chain_count : 1;
        input_channel_count : 1;
        output_channel_count : 1;
        Connections +{
         EdtChannelsIn(1) {
         }
         EdtChannelsOut(1) {
         }
        }
    }
  }
}

set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsIn(1) [get_single_name [get_auxiliary_pins net_in_1[7] -direction input]]
set_config_value port_pin_name \
-in $spec/EDT/Controller(c1)/Connections/EdtChannelsOut(1) [get_single_name [get_auxiliary_pins net_out[7] -direction output] ]
report_config_data $spec

#set_config_value $spec/use_rtl_synchronizer_cell On

# Generate and insert the hardware
process_dft_specification

# Extract IJAG network and create ICL file for the design
extract_icl

# Synthesize the hardware created in this pass using DC-Shell
run_synthesis
```

### 6_insert_scan

来源：[6_insert_scan](E:/tessent/Reference_2026/tessent/hier_flow/flow/6_insert_scan)。

```tcl
set top top

set_context dft -scan -design_id scan
set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_design $top -design_id edt -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose

set_current_design $top

set_system_mode analysis
report_scan_segments
report_clocks


set edt_instance [get_name_list [get_instance -of_module [get_name [get_icl_module -of_instances top* -filter tessent_instrument_type==mentor::edt]]]]
add_scan_mode edt_mode -edt_instance $edt_instance

analyze_scan_chains
report_scan_chains

insert_test_logic
```

### 7_atpg

来源：[7_atpg](E:/tessent/Reference_2026/tessent/hier_flow/flow/7_atpg)。

```tcl
set top top

set_context patterns -scan

set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_design $top -design_id scan -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose

set_current_design $top

set_current_mode edt_top_stuck

import_scan_mode edt_mode

report_core_instances

set_static_dft_signal_values tck_occ_en 1
report_static_dft_signal_settings

report_clocks

set_system_mode analysis
report_clocks

# Generate patterns
create_patterns
report_statistics -detail

# Store TCD, flat_model, fault list and patDB format files in the TSDB directory
write_tsdb_data -replace

# Write Verilog patterns for simulation
write_patterns ./patterns/${top}_stuck_parallel.v -verilog -parallel -replace -scan -parameter_list {SIM_KEEP_PATH 1}
set_pattern_filtering -sample_per_type 2
write_patterns ./patterns/${top}_stuck_serial.v -verilog -serial -replace -parameter_list {SIM_KEEP_PATH 1}

write_patterns ./patterns/${top}_stuck.stil -stil -replace

read_faults -module add_ram -mode int_mode_sa -fault_type stuck -merge -graybox
read_faults -module add_ram_inv -mode int_mode_sa -fault_type stuck -merge -graybox
report_statistics -detail
```

### 8_retargeting1

来源：[8_retargeting1](E:/tessent/Reference_2026/tessent/hier_flow/flow/8_retargeting1)。

```tcl
set top top

set_context patterns -scan_retargeting

set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_design $top -design_id scan -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose

set_current_design $top

set_current_mode retarget1_stuck
set_static_dft_signal_values retargeting1_mode 1
add_core_instances -instances {add_ram_1 add_ram_2} -core add_ram -mode int_mode_sa

report_core_descriptions
report_clocks

set_system_mode analysis
write_tsdb_data -replace

read_patterns -module add_ram -fault_type stuck -mode int_mode_sa
set_external_capture_options -pll_cycles 5 [lindex [get_timeplate_list] 0]

write_patterns ./patterns/add_ram_stuck.stil -stil -replace
```

### 9_retargeting2

来源：[9_retargeting2](E:/tessent/Reference_2026/tessent/hier_flow/flow/9_retargeting2)。

```tcl
set top top

set_context patterns -scan_retargeting

set_tsdb_output_directory ./tsdb

open_tsdb ./2_add_ram/tsdb
open_tsdb ./3_add_ram_inv/tsdb

read_cell_library ../lib/adk.tcelllib
read_cell_library ../lib/picdram.tcelllib
set_design_sources -format tcd_memory -Y ../lib -extension tcd_memory
read_design $top -design_id scan -verbose
read_design add_ram -design_id scan -view graybox -verbose
read_design add_ram_inv -design_id scan -view graybox -verbose

set_current_design $top

set_current_mode retarget2_stuck
set_static_dft_signal_values retargeting2_mode 1
add_core_instances -instances {add_ram_inv_1} -core add_ram_inv -mode int_mode_sa

report_core_descriptions
report_clocks

set_system_mode analysis
write_tsdb_data -replace

read_patterns -module add_ram_inv -fault_type stuck -mode int_mode_sa
set_external_capture_options -pll_cycles 5 [lindex [get_timeplate_list] 0]

write_patterns ./patterns/add_ram_stuck.stil -stil -replace
```
