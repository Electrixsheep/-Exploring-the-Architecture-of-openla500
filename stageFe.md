
## 取指阶段（Fetch Stage）解读

取指阶段是五级流水线的第一级，负责从指令缓存中获取指令并为后续译码阶段准备数据。取指阶段包含预取指（pre-IF）和取指（IF）两个子阶段，实现了复杂的地址生成、分支预测、例外处理和流水线控制逻辑。

### 1. 取指到译码阶段信号总线设计

取指阶段到译码阶段的总线信号定义语句如下所示，这实际上是一种把信号"打包"的做法，这样的Verilog写法，简化了例化时的接口，并且方便后续扩展信号（只需要增宽总线即可)
>btb_xx是分支预测器相关的信号，分支预测器不是《设计实战》中要求必须实现的，但是，是高性能处理器的必备组件阶段到译码阶段的总线信号定义语句如下所示，这实际上是一种把信号”打包“的做法，这样的Verilog写法，简化了例化时的接口，并且方便后续扩展信号（只需要增宽总线即可)
>btb_xx是分支预测器相关的信号，分支预测器不是《设计实战》中要求必须实现的，但是，是高性能处理器的必备组件

```verilog
assign fs_to_ds_bus = {btb_ret_pc_t,    //108:77  分支预测器返回的目标PC
                       btb_index_t,     //76:72   BTB表项索引  
                       btb_taken_t,     //71:71   分支是否被预测为taken
                       btb_en_t,        //70:70   BTB预测是否有效
                       //用于性能计数器统计icache的命中率，
                       icache_miss,     //69:69   指令Cache未命中标志
						//例外有效信号
                       excp,            //68:68   取指阶段例外发生标志
						//用于表明发生了哪种例外信号
                       excp_num,        //67:64   例外类型编码
						//取出的指令大小
                       fs_inst,         //63:32   从Cache或缓存中读取的32位指令
						//所取出指令对应的pc值
                       fs_pc            //31:0    当前指令的PC值
                      };
```

### 2. 流水线冲刷状态机设计

以下是if级流水线冲刷状态机的写法，用来辅助实现冲刷逻辑
>流水线冲刷是CPU中会发生的一种常见情况，在具有实用意义的CPU中，各级的流水线冲刷逻辑是必备的，如分支预测错误、发生例外等等，都会需要冲刷流水线。实现CPU的话，最好一开始就思考如何在各个流水级，都分别如何进行流水线的冲刷，以免手忙脚乱
```verilog
```verilog
//等号右侧：可能的刷新信号来源，包括各种需要重新取指的情况
assign flush_sign = ertn_flush || excp_flush || refetch_flush || icacop_flush || idle_flush;
//冲刷状态机主要是考虑以下特殊情况而设计：接到刷新信号，表明此处需要冲刷，但是不应该马上进行时，就需要"把冲刷这件事记下来"，以免"遗忘"（下一拍输入的冲刷信号将会消失）。称以上情况为延迟冲刷，冲刷状态机本质上是"记录是否有待执行的冲刷"
//以下两种情况需要进行延迟冲刷1.当刷新信号到来，需要重新取指，但是icache不接收新请求的时候，此时冲刷信号并不能真正作用，需要暂缓处理2.处理器执行了idle指令而停止工作（等待唤醒）的时候
//否则，将会立即冲刷，立即冲刷不会对状态机产生影响(因为冲刷状态机是考虑延迟冲刷设计的)
assign flush_inst_delay = flush_sign && !inst_addr_ok || idle_flush;      //需要延迟冲刷的条件
assign flush_inst_go_dirt = flush_sign && inst_addr_ok && !idle_flush;    //可以立即冲刷的条件
//flush state machine - 冲刷状态机实现
reg [31:0] flush_inst_req_buffer;    //缓存待冲刷时需要跳转的PC值
reg        flush_inst_req_state;     //状态机当前状态
//定义状态机状态，方便阅读和维护代码
localparam flush_inst_req_empty = 1'b0;    //空闲状态：无待处理的冲刷请求
localparam flush_inst_req_full  = 1'b1;    //忙碌状态：有待处理的冲刷请求
always @(posedge clk) begin
    if (reset) begin
        flush_inst_req_state <= flush_inst_req_empty;
    end 
    else case (flush_inst_req_state)
        flush_inst_req_empty: begin
        //冲刷状态机中不存在"记下的冲刷"（即空白）时，如果遇到了要记下来的冲刷（延迟冲刷），就把冲刷需要的信息（pc）"写在冲刷状态机这个记事本"
            if(flush_inst_delay) begin
                flush_inst_req_buffer <= nextpc;               //保存需要跳转的目标PC
                flush_inst_req_state  <= flush_inst_req_full;  //转换到忙碌状态
            end
        end
        //冲刷状态机中已经写上了待处理的冲刷，如果看到预取指阶段的信号ready了，说明这个冲刷可以处理了，将"擦掉"冲刷记录回到空白状态
        flush_inst_req_full: begin
            if(pfs_ready_go) begin
                flush_inst_req_state  <= flush_inst_req_empty; //冲刷完成，回到空闲状态
            end
        //已经有记录，但是有新的冲刷信号到来的时候，需要将待处理的冲刷记录更新到最新
            else if (flush_sign) begin
                flush_inst_req_buffer <= nextpc;               //更新为最新的冲刷目标PC
            end
        end
    endcase
end

```

### 3. 预取指阶段逻辑与PC生成

以下是pre-if阶段的逻辑，预取指阶段的作用，主要是生成访问icache的信号（即nextpc）
>nextpc值主要来源：中断和例外，顺序pc，跳转指令pc，分支预测器

```
以下是pre-if阶段的逻辑，预取指阶段的作用，主要是生成访问icache的信号（即nextpc）
>nextpc值主要来源：中断和例外，顺序pc，跳转指令pc，分支预测器
```verilog
```verilog
// pre-IF stage - 预取指阶段逻辑
// 当可以取指，或者pre-if阶段有例外（这时候直接取，方便处理例外），并且icache允许进入时，pre-if就会go
assign pfs_ready_go = (inst_valid || pfs_excp) && inst_addr_ok;
assign to_fs_valid  = ~reset && pfs_ready_go;
//下一个顺序pc值，标准的PC+4递增逻辑
assign seq_pc       =  fs_pc + 32'h4;
//例外发生时，将跳转到的pc值，tlb重填例外对应不同的例外入口
//TLB重填例外有专门的入口地址，其他例外使用通用例外入口
assign excp_entry   = {32{excp_tlbrefill}}  & csr_tlbrentry |    //TLB重填例外入口
                      {32{!excp_tlbrefill}} & csr_eentry    ;    //通用例外入口
//冲刷时候，将跳转到的pc值，异常处理导致的冲刷，将跳转到异常处理返回地址开始执行，否则将从提交的指令的下一条开始重新执行
assign inst_flush_pc = {32{ertn_flush}}                         & csr_era         |    //ERTN指令返回地址
                       {32{refetch_flush || icacop_flush || idle_flush}} 
										                       & (ws_pc + 32'h4) ;   //其他冲刷：从WB级PC+4重新开始
//各种情况下的nextpc值：按优先级从上到下依次是：1.有之前未处理的冲刷时2.有例外冲刷时3.其他冲刷时4.等待分支预测计算目标5.分支预测错误冲刷（注意，这里与上fs_valid，表明有可能存在：分支预测错误，但实际上预测错的指令没有影响（即取指阶段的指令是无效的），则不属于此种情况）6.常规分支预测跳转7.都不满足时，取顺序pc值
assign nextpc = (flush_inst_req_state == flush_inst_req_full) ? flush_inst_req_buffer:      //优先级最高：处理缓存的冲刷请求  
	                 excp_flush                               ? excp_entry           :      //例外冲刷：跳转到例外处理入口
                (ertn_flush || refetch_flush || icacop_flush || idle_flush)
												             ? inst_flush_pc        :      //其他类型冲刷：跳转到指定地址
                (br_target_inst_req_state == br_target_inst_req_wait_br_target) 
										                 ? br_target_inst_req_buffer : //等待分支目标计算完成
                btb_pre_error_flush && fs_valid           ? btb_pre_error_flush_target: //分支预测错误冲刷
                fetch_btb_target                          ? btb_ret_pc_t              : //分支预测器给出的目标地址                                                                      
                                                            seq_pc;                     //默认：顺序PC+4
```

### 4. 指令获取控制逻辑

```verilog
/*
*当遇到tlb异常时，停止指令取 fetch，直到异常清除。避免取无用的指令。
*但在btb状态机或清除状态机工作时不应锁定。
*/
assign tlb_excp_lock_pc = tlb_excp_cancel_req && br_target_inst_req_state != br_target_inst_req_wait_br_target && flush_inst_req_state != flush_inst_req_full;
//指令获取有效信号：综合考虑各种stall和冲刷条件
assign inst_valid = (fs_allowin && !pfs_excp && !tlb_excp_lock_pc || flush_sign || btb_pre_error_flush) && !(idle_flush || idle_lock);
//取指的参数配置 - 指令Cache访问参数
assign inst_op     = 1'b0;      //读操作
assign inst_wstrb  = 4'h0;      //无写字节使能（只读）
assign inst_addr   = nextpc;    //访问地址为nextpc
//指令cache对cpu没有写接口，设置全0省功耗
assign inst_wdata  = 32'b0;     //写数据恒为0
//取出指令之后，如果译码阶段不能立即接受指令（stall了），则需要将该处理而未处理的指令也"记下来"，并优先处理之
//当复位/取指和译码握手成功，即指令可以正常发射到译码/冲刷时，应处理而未处理的指令被"一笔勾销"
//理论上来说，遇到这种情况，也可以把pc发回pre-if重新取指，但是控制逻辑会复杂很多
assign fs_inst     = (inst_buff_enable) ? inst_rd_buff : inst_rdata;  //优先使用缓存指令
//指令缓存机制：当Cache返回数据但译码级暂时无法接收时暂存指令
always @(posedge clk) begin
    if (reset || (fs_ready_go && ds_allowin) || flush_sign) begin
        inst_buff_enable  <= 1'b0;    //清除缓存标志
    end
    else if ((inst_data_ok) && !ds_allowin) begin
        inst_rd_buff <= inst_rdata;   //保存Cache返回的指令
        inst_buff_enable  <= 1'b1;    //置位缓存有效标志
    end
end
```

### 5. 例外检测与地址转换逻辑

各种例外信号，以及页表相关信号如下所示：
```

```verilog
/*
*当遇到tlb异常时，停止指令取 fetch，直到异常清除。避免取无用的指令。
*但在btb状态机或清除状态机工作时不应锁定。
*/
assign tlb_excp_lock_pc = tlb_excp_cancel_req && br_target_inst_req_state != br_target_inst_req_wait_br_target && flush_inst_req_state != flush_inst_req_full;
assign inst_valid = (fs_allowin && !pfs_excp && !tlb_excp_lock_pc || flush_sign || btb_pre_error_flush) && !(idle_flush || idle_lock);
//取指的参数配置
assign inst_op     = 1'b0;
assign inst_wstrb  = 4'h0;
assign inst_addr   = nextpc; //nextpc
//指令cache对cpu没有写接口，设置全0省功耗
assign inst_wdata  = 32'b0;
//取出指令之后，如果译码阶段不能立即接受指令（stall了），则需要将该处理而未处理的指令也“记下来”，并优先处理之
//当复位/取指和译码握手成功，即指令可以正常发射到译码/冲刷时，应处理而未处理的指令被“一笔勾销”
//理论上来说，遇到这种情况，也可以把pc发回pre-if重新取指，但是控制逻辑会复杂很多
assign fs_inst     = (inst_buff_enable) ? inst_rd_buff : inst_rdata;
always @(posedge clk) begin
    if (reset || (fs_ready_go && ds_allowin) || flush_sign) begin
        inst_buff_enable  <= 1'b0;
    end
    else if ((inst_data_ok) && !ds_allowin) begin
        inst_rd_buff <= inst_rdata;
        inst_buff_enable  <= 1'b1;
    end
end
```
各种例外信号，以及页表相关信号如下所示：
```verilog
//pc非对齐例外信号 - 指令地址必须4字节对齐
assign pfs_excp_adef = (nextpc[0] || nextpc[1]); //word align，检查低2位是否为0
//tlb相关例外，在打开页表翻译模式的前提下可能触发
assign fs_excp_tlbr = !inst_tlb_found && inst_addr_trans_en;        //TLB重填例外：TLB中未找到对应表项
assign fs_excp_pif  = !inst_tlb_v && inst_addr_trans_en;            //页面无效例外：TLB表项有效位为0
assign fs_excp_ppi  = (csr_plv > inst_tlb_plv) && inst_addr_trans_en; //页特权级例外：当前特权级低于页面要求的特权级
assign tlb_excp_cancel_req = fs_excp_tlbr || fs_excp_pif || fs_excp_ppi; //TLB例外时取消Cache请求

//pre-if的例外大全，目前只有指令地址非对齐例外
assign pfs_excp = pfs_excp_adef;
assign pfs_excp_num = {pfs_excp_adef};
//取指阶段的例外大全，汇总所有可能的例外
assign excp = fs_excp || fs_excp_tlbr || fs_excp_pif || fs_excp_ppi ;
assign excp_num = {fs_excp_ppi, fs_excp_pif, fs_excp_tlbr, fs_excp_num};
//页表翻译模式打开判别：分页模式且不使用DMW窗口时需要TLB转换
assign inst_addr_trans_en = pg_mode && !dmw0_en && !dmw1_en;
//addr dmw trans  //TOT - DMW（直接映射窗口）地址转换使能
//DMW0窗口使能：检查特权级匹配和虚拟地址段匹配
assign dmw0_en = ((csr_dmw0[`PLV0] && csr_plv == 2'd0) || (csr_dmw0[`PLV3] && csr_plv == 2'd3)) && (fs_pc[31:29] == csr_dmw0[`VSEG]) && pg_mode;
//DMW1窗口使能：检查特权级匹配和虚拟地址段匹配
assign dmw1_en = ((csr_dmw1[`PLV0] && csr_plv == 2'd0) || (csr_dmw1[`PLV3] && csr_plv == 2'd3)) && (fs_pc[31:29] == csr_dmw1[`VSEG]) && pg_mode;
//uncache judgement - 非缓存访问判断
assign da_mode = csr_da && !csr_pg;  //直接地址模式
assign pg_mode = csr_pg && !csr_da;  //分页模式
//指令Cache绕过条件：多种情况下需要直接访问内存而不经过Cache
assign inst_uncache_en = (da_mode && (csr_datf == 2'b0))                 ||  //直接地址模式且MAT为0
                         (dmw0_en && (csr_dmw0[`DMW_MAT] == 2'b0))       ||  //DMW0窗口且MAT为0
                         (dmw1_en && (csr_dmw1[`DMW_MAT] == 2'b0))       ||  //DMW1窗口且MAT为0
                         (inst_addr_trans_en && (inst_tlb_mat == 2'b0))  ||  //TLB转换且MAT为0
                         disable_cache;                                       //强制禁用Cache标志
//assign inst_uncache_en = 1'b1; //used for debug - 调试时可强制禁用Cache
```

### 6. IF阶段流水线控制逻辑

以下是if阶段的逻辑，流水线寄存器维护的数据有pc和例外相关数据，指令直接从待处理指令的缓存或者icache中得到（无需再寄存到流水线寄存器中，不然会卡一拍）
```verilog
```verilog
// IF stage - 取指阶段流水线控制

//icache取出了指令/已经存有待处理的指令/在例外状态，都说明取指阶段有数据供流通到下一级（例外状态有点类似"假"有数据）
assign fs_ready_go    = inst_data_ok || inst_buff_enable || excp;
//《设计实战》上的标准流水线控制信号
assign fs_allowin     = !fs_valid || fs_ready_go && ds_allowin;      //取指级允许新数据进入的条件
assign fs_to_ds_valid =  fs_valid && fs_ready_go;                    //向译码级发送有效数据的条件
//取指阶段流水线寄存器更新逻辑
always @(posedge clk) begin
    if (reset || flush_inst_delay) begin
        fs_valid <= 1'b0;                                            //复位或延迟冲刷时清除valid
    end
    else if (fs_allowin) begin
        fs_valid <= to_fs_valid;                                     //标准流水线valid传递
    end
    if (reset) begin
        fs_pc        <= 32'h1bfffffc;  //trick: to make nextpc be 0x1c000000 during reset - 复位技巧
        fs_excp      <= 1'b0;                                        //清除例外标志
        fs_excp_num  <= 4'b0;                                        //清除例外编号
    end
    //立即冲刷时，无需等待取指阶段allowin，直接更新寄存器
    else if (to_fs_valid && (fs_allowin || flush_inst_go_dirt)) begin
        fs_pc        <= nextpc;                                      //更新PC值
        fs_excp      <= pfs_excp;                                    //传递例外标志
        fs_excp_num  <= pfs_excp_num;                               //传递例外编号
    end
end
//前往btb以及tlb的信号
assign fetch_pc  = nextpc;                                           //向分支预测器和TLB提供PC
assign fetch_en  = inst_valid && inst_addr_ok;                      //取指使能信号
endmodule

## 取指阶段设计要点总结

### 1. 两级取指结构
- **预取指阶段（pre-IF）**：负责生成下一个PC地址，处理各种跳转逻辑
- **取指阶段（IF）**：负责从指令Cache获取指令，维护流水线寄存器

### 2. 复杂的PC生成逻辑
按优先级处理多种PC来源：
1. 冲刷状态机中缓存的PC（最高优先级）
2. 例外处理PC
3. 其他类型冲刷PC
4. 分支预测器提供的目标PC
5. 顺序PC+4（默认情况）

### 3. 冲刷状态机设计
解决了取指阶段面临的关键问题：
- **延迟冲刷**：当需要冲刷但无法立即执行时缓存冲刷请求
- **状态管理**：通过简单的两状态机管理冲刷请求的生命周期
- **请求更新**：支持在等待期间更新冲刷目标地址

### 4. 指令缓存机制
- 当指令Cache返回数据但译码阶段暂时无法接收时暂存指令
- 避免了重复的Cache访问，提高了流水线效率
- 简化了控制逻辑，避免了复杂的PC回退机制

### 5. 地址转换与例外处理
- 支持直接地址模式（DA）和分页模式（PG）
- 实现DMW窗口机制加速地址转换
- 完整的TLB例外检测：重填、页无效、特权级违例
- PC对齐检查确保指令地址的合法性

取指阶段作为流水线的源头，其设计质量直接影响整个处理器的性能和正确性。openLA500的取指阶段实现了现代处理器所需的所有关键特性，为后续流水级提供了稳定可靠的指令流。
```





