主要对if_stage.v中，非分支预测器相关的代码进行解读

取指阶段到译码阶段的总线信号定义语句如下所示，这实际上是一种把信号”打包“的做法，这样的Verilog写法，简化了例化时的接口，并且方便后续扩展信号（只需要增宽总线即可)
>btb_xx是分支预测器相关的信号，分支预测器不是《设计实战》中要求必须实现的，但是，是高性能处理器的必备组件

```verilog
assign fs_to_ds_bus = {btb_ret_pc_t,    //108:77
                       btb_index_t,     //76:72
                       btb_taken_t,     //71:71
                       btb_en_t,        //70:70
                       //用于性能计数器统计icache的命中率，
                       icache_miss,     //69:69
						//例外有效信号
                       excp,            //68:68
						//用于表明发生了哪种例外信号
                       excp_num,        //67:64
						//取出的指令大小
                       fs_inst,         //63:32
						//所取出指令对应的pc值
                       fs_pc            //31:0
                      };
```
以下是if级流水线冲刷状态机的写法，用来辅助实现冲刷逻辑
>流水线冲刷是CPU中会发生的一种常见情况，在具有实用意义的CPU中，各级的流水线冲刷逻辑是必备的，如分支预测错误、发生例外等等，都会需要冲刷流水线。实现CPU的话，最好一开始就思考如何在各个流水级，都分别如何进行流水线的冲刷，以免手忙脚乱
```verilog
//等号右侧：可能的刷新信号来源
assign flush_sign = ertn_flush || excp_flush || refetch_flush || icacop_flush || idle_flush;
//冲刷状态机主要是考虑以下特殊情况而设计：接到刷新信号，表明此处需要冲刷，但是不应该马上进行时，就需要“把冲刷这件事记下来”，以免“遗忘”（下一拍输入的冲刷信号将会消失）。称以上情况为延迟冲刷，冲刷状态机本质上是“记录是否有待执行的冲刷”
//以下两种情况需要进行延迟冲刷1.当刷新信号到来，需要重新取指，但是icache不接收新请求的时候，此时冲刷信号并不能真正作用，需要暂缓处理2.处理器执行了idle指令而停止工作（等待唤醒）的时候
//否则，将会立即冲刷，立即冲刷不会对状态机产生影响(因为冲刷状态机是考虑延迟冲刷设计的)
assign flush_inst_delay = flush_sign && !inst_addr_ok || idle_flush;
assign flush_inst_go_dirt = flush_sign && inst_addr_ok && !idle_flush;
//flush state machine
reg [31:0] flush_inst_req_buffer;
reg        flush_inst_req_state;
//定义状态机状态，方便阅读和维护代码
localparam flush_inst_req_empty = 1'b0;
localparam flush_inst_req_full  = 1'b1;
always @(posedge clk) begin
    if (reset) begin
        flush_inst_req_state <= flush_inst_req_empty;
    end 
    else case (flush_inst_req_state)
        flush_inst_req_empty: begin
        //冲刷状态机中不存在“记下的冲刷”（即空白）时，如果遇到了要记下来的冲刷（延迟冲刷），就把冲刷需要的信息（pc）"写在冲刷状态机这个记事本"
            if(flush_inst_delay) begin
                flush_inst_req_buffer <= nextpc;
                flush_inst_req_state  <= flush_inst_req_full;
            end
        end
        //冲刷状态机中已经写上了待处理的冲刷，如果看到预取指阶段的信号ready了，说明这个冲刷可以处理了，将“擦掉”冲刷记录回到空白状态
        flush_inst_req_full: begin
            if(pfs_ready_go) begin
                flush_inst_req_state  <= flush_inst_req_empty;
            end
        //已经有记录，但是有新的冲刷信号到来的时候，需要将待处理的冲刷记录更新到最新
            else if (flush_sign) begin
                flush_inst_req_buffer <= nextpc;
            end
        end
    endcase
end

```
以下是pre-if阶段的逻辑，预取指阶段的作用，主要是生成访问icache的信号（即nextpc）
>nextpc值主要来源：中断和例外，顺序pc，跳转指令pc，分支预测器
```verilog
// pre-IF stage
// 当可以取指，或者pre-if阶段有例外（这时候直接取，方便处理例外），并且icache允许进入时，pre-if就会go
assign pfs_ready_go = (inst_valid || pfs_excp) && inst_addr_ok;
assign to_fs_valid  = ~reset && pfs_ready_go;
//下一个顺序pc值
assign seq_pc       =  fs_pc + 32'h4;
//例外发生时，将跳转到的pc值，tlb重填例外对应不同的例外入口
assign excp_entry   = {32{excp_tlbrefill}}  & csr_tlbrentry |
                      {32{!excp_tlbrefill}} & csr_eentry    ;
//冲刷时候，将跳转到的pc值，异常处理导致的冲刷，将跳转到异常处理返回地址开始执行，否则将从提交的指令的下一条开始重新执行
assign inst_flush_pc = {32{ertn_flush}}                                  & csr_era         |
                       {32{refetch_flush || icacop_flush || idle_flush}} & (ws_pc + 32'h4) ;
//各种情况下的nextpc值：按优先级从上到下依次是：1.有之前未处理的冲刷时2.有例外冲刷时3.其他冲刷时4.等待分支预测计算目标5.分支预测错误冲刷（注意，这里与上fs_valid，表明有可能存在：分支预测错误，但实际上预测错的指令没有影响（即取指阶段的指令是无效的），则不属于此种情况）6.常规分支预测跳转7.都不满足时，取顺序pc值
assign nextpc = (flush_inst_req_state == flush_inst_req_full)                   ? flush_inst_req_buffer     :
                excp_flush                                                      ? excp_entry                :
                (ertn_flush || refetch_flush || icacop_flush || idle_flush)     ? inst_flush_pc             :
                (br_target_inst_req_state == br_target_inst_req_wait_br_target) ? br_target_inst_req_buffer :
                btb_pre_error_flush && fs_valid                                 ? btb_pre_error_flush_target:
                fetch_btb_target                                                ? btb_ret_pc_t              :
                                                                                  seq_pc                    ;
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
//pc非对齐例外信号
assign pfs_excp_adef = (nextpc[0] || nextpc[1]); //word align
//tlb相关例外，在打开页表翻译模式的前提下可能触发
assign fs_excp_tlbr = !inst_tlb_found && inst_addr_trans_en;
assign fs_excp_pif  = !inst_tlb_v && inst_addr_trans_en;
assign fs_excp_ppi  = (csr_plv > inst_tlb_plv) && inst_addr_trans_en;
assign tlb_excp_cancel_req = fs_excp_tlbr || fs_excp_pif || fs_excp_ppi;

//pre-if的例外大全，目前只有指令地址非对齐例外
assign pfs_excp = pfs_excp_adef;
assign pfs_excp_num = {pfs_excp_adef};
//取指阶段的例外大全
assign excp = fs_excp || fs_excp_tlbr || fs_excp_pif || fs_excp_ppi ;
assign excp_num = {fs_excp_ppi, fs_excp_pif, fs_excp_tlbr, fs_excp_num};
//页表翻译模式打开判别
assign inst_addr_trans_en = pg_mode && !dmw0_en && !dmw1_en;
//addr dmw trans  //TOT
assign dmw0_en = ((csr_dmw0[`PLV0] && csr_plv == 2'd0) || (csr_dmw0[`PLV3] && csr_plv == 2'd3)) && (fs_pc[31:29] == csr_dmw0[`VSEG]) && pg_mode;
assign dmw1_en = ((csr_dmw1[`PLV0] && csr_plv == 2'd0) || (csr_dmw1[`PLV3] && csr_plv == 2'd3)) && (fs_pc[31:29] == csr_dmw1[`VSEG]) && pg_mode;
//uncache judgement
assign da_mode = csr_da && !csr_pg;
assign pg_mode = csr_pg && !csr_da;
assign inst_uncache_en = (da_mode && (csr_datf == 2'b0))                 ||
                         (dmw0_en && (csr_dmw0[`DMW_MAT] == 2'b0))       ||
                         (dmw1_en && (csr_dmw1[`DMW_MAT] == 2'b0))       ||
                         (inst_addr_trans_en && (inst_tlb_mat == 2'b0))  ||
                         disable_cache;
//assign inst_uncache_en = 1'b1; //used for debug
```
以下是if阶段的逻辑，流水线寄存器维护的数据有pc和例外相关数据，指令直接从待处理指令的缓存或者icache中得到（无需再寄存到流水线寄存器中，不然会卡一拍）
```verilog
// IF stage

//icache取出了指令/已经存有待处理的指令/在例外状态，都说明取指阶段有数据供流通到下一级（例外状态有点类似“假”有数据）
assign fs_ready_go    = inst_data_ok || inst_buff_enable || excp;
//《设计实战》上的标准流水线控制信号
assign fs_allowin     = !fs_valid || fs_ready_go && ds_allowin;
assign fs_to_ds_valid =  fs_valid && fs_ready_go;
always @(posedge clk) begin
    if (reset || flush_inst_delay) begin
        fs_valid <= 1'b0;
    end
    else if (fs_allowin) begin
        fs_valid <= to_fs_valid;
    end
    if (reset) begin
        fs_pc        <= 32'h1bfffffc;  //trick: to make nextpc be 0x1c000000 during reset 
        fs_excp      <= 1'b0;
        fs_excp_num  <= 4'b0;
    end
    //立即冲刷时，无需等待取指阶段allowin
    else if (to_fs_valid && (fs_allowin || flush_inst_go_dirt)) begin
        fs_pc        <= nextpc;
        fs_excp      <= pfs_excp;
        fs_excp_num  <= pfs_excp_num;
    end
end
//前往btb以及tlb
assign fetch_pc  = nextpc;
assign fetch_en  = inst_valid && inst_addr_ok;
endmodule
```





