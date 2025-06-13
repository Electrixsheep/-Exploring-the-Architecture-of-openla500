dcache_req_or_inst_en是一条重要的控制信号，反映了当前周期，执行级的指令是否应该
1.产生效果，主要体现在其控制一系列访存（即对cache产生影响）的操作中，openla500的实现中，这些操作会在执行级产生影响，所以需要该控制信号，防止有多余/错误的影响（即，按理来说“不应该”产生的影响产生了）。dcache_req_or_inst_en能控制的信号包括：
- data_valid
- icacop_op_en/dcacop_op_en
- preld_en
2.往后流水，dcache_req_or_inst_en是执行级ready逻辑的重要部分
```verilog
assign dcache_req_or_inst_en = es_valid && !excp && ms_allowin && !es_flush_sign && !ms_flush;
```

执行阶段往访存阶段传递的信号总线如下所示
注意"es_"开头的信号，是从前面的流水级中生成并往后传递的，这部分信号的解读不再重复
```verilog
assign es_to_ms_bus = {es_csr_data      ,  //424:393  for difftest
                       es_csr_rstat_en  ,  //392:392  for difftest
                       data_wdata       ,  //391:360  for difftest
                       es_inst_st_en    ,  //359:352  for difftest
                       data_addr        ,  //351:320  for difftest
                       es_inst_ld_en    ,  //319:312  for difftest
                       es_cnt_inst      ,  //311:311  for difftest
                       es_timer_64      ,  //310:247  for difftest
                       es_inst          ,  //246:215  for difftest
//error_va = pv_addr = es_alu_result                       
//与访存相关的例外和ale例外，需要将错误的访存虚拟地址传至WB进行例外和页表相关处理                      
                       error_va         ,  //214:183
                       es_idle          ,  //182:182
                       es_cacop         ,  //181:181
//preld_inst = es_preld && ((preld_hint == 5'd0) || (preld_hint == 5'd8)) 
//preld指令定义中，字段hint为0/8时，prled才有效，判定hint值是否有效的工作在执行级完成的(译码级只传递了是否为prled指令)
                       preld_inst       ,  //180:180
                       es_br_pre_error  ,  //179:179
                       es_br_pre        ,  //178:178
                       es_icache_miss   ,  //177:177
                       es_br_inst       ,  //176:176
                       icacop_op_en     ,  //175:175
                       es_mem_sign_exted,  //174:174  //only add this, not used. 
                       es_invtlb_vpn    ,  //173:155
                       es_invtlb_asid   ,  //154:145
                       es_invtlb        ,  //144:144
                       es_tlbrd         ,  //143:143
                       es_refetch       ,  //142:142
                       es_tlbfill       ,  //141:141
                       es_tlbwr         ,  //140:140
                       es_tlbsrch       ,  //139:139
                       es_store_op      ,  //138:138
                       es_sc_w          ,  //137:137
                       es_ll_w          ,  //136:136
                       excp_num         ,  //135:126
                       es_csr_we        ,  //125:125
                       es_csr_idx       ,  //124:111
                       es_csr_result    ,  //110:79
                       es_ertn          ,  //78:78
                       excp             ,  //77:77
                       es_mem_size      ,  //76:75
                       es_mul_div_op    ,  //74:71
                       es_load_op       ,  //70:70
                       es_gr_we         ,  //69:69
                       es_dest          ,  //68:64
//执行级将计算出指令要写回的结果，往后流水级传递                       
                       exe_result       ,  //63:32
                       es_pc               //31:0
                      };

```
流水线控制信号如下所示
```verilog
//传递到下一流水级的控制信号
assign es_to_ds_valid = es_valid;
//判定需要访存的信号
assign access_mem = es_load_op || es_store_op;
//执行阶段的冲刷流水线信号（wb发出，即《设计实战》中提到的重取指类型的冲刷
assign es_flush_sign  = excp_flush || ertn_flush || refetch_flush || icacop_flush || idle_flush;
//需要发出icacop操作，但是icache繁忙的时候将会stall
assign icacop_inst_stall = icacop_op_en && !icache_unbusy;
//同时满足以下几种情况，执行级才算ready：
//1.除法器操作 
//2.无需访问dcache，或者dcache握手成功的时候
//3.如果是tlbsrch指令，无冒险时
//4.如果是icacop指令，icache不busy时
//5.如果例外发生，无条件ready（方便例外处理流程）
assign es_ready_go    = (!div_stall && ((dcache_req_or_inst_en && data_addr_ok) || !(access_mem || dcacop_inst || preld_inst)) && !tlbsrch_stall && !icacop_inst_stall) || excp;
//公式的allowin
assign es_allowin     = !es_valid || es_ready_go && ms_allowin;
//公式的对下一级valid
assign es_to_ms_valid =  es_valid && es_ready_go;
//标准流水线valid信号写法
always @(posedge clk) begin
    if (reset || es_flush_sign) begin     
        es_valid <= 1'b0;
    end
    else if (es_allowin) begin 
        es_valid <= ds_to_es_valid;
    end
    if (ds_to_es_valid && es_allowin) begin
        ds_to_es_bus_r <= ds_to_es_bus;
    end
end
```
ALU输入逻辑生成
```verilog
//生成加法器的两个源操作数
assign es_alu_src1 = es_src1_is_pc ? es_pc : es_rj_value;
assign es_alu_src2 = (es_src2_is_imm) ? es_imm : 
                     (es_src2_is_4)   ? 32'd4  : es_rkd_value;
//调用加法器，生成组合逻辑结果
alu u_alu(
    .alu_op     (es_alu_op    ),
    .alu_src1   (es_alu_src1  ),  //bug3 es_alu_src2
    .alu_src2   (es_alu_src2  ),
    .alu_result (es_alu_result)
    );                     
//生成exe结果，csr读出结果在执行级得到
assign exe_result = es_res_from_csr ? es_csr_data : es_alu_result;
```

乘/除法器逻辑，其结果直接输入访存级，执行级主要是相关控制逻辑
```verilog
//乘除法器使能逻辑
assign es_div_enable = (es_mul_div_op[2] | es_mul_div_op[3]) & es_valid;
assign es_mul_enable = es_mul_div_op[0] | es_mul_div_op[1];
//除法未完成，等待计算完成
assign div_stall     = es_div_enable & ~div_complete;
```

前递通路比较标准的写法，其中stall信号需要看具体的流水级决定哪些操作需要stall
```verilog
//forward path标准写法
assign dest_zero            = (es_dest == 5'b0); 
assign forward_enable       = es_gr_we & ~dest_zero & es_valid;
//无法在执行级当拍得到结果，需要stall
assign dep_need_stall       = es_load_op | es_div_enable | es_mul_enable;
assign es_to_ds_forward_bus = {dep_need_stall ,  //38:38
                               forward_enable ,  //37:37
                               es_dest        ,  //36:32
                               exe_result         //31:0
                              };
```
执行级与csr相关的操作主要是csr指令得到csr读出结果，写到通用寄存器中
```verilog
//访问csr相关信号
//csr mask
assign csr_mask_result = (es_rj_value & es_rkd_value) | (~es_rj_value & es_csr_data);
assign es_csr_result   = es_csr_mask ? csr_mask_result : es_rkd_value;

```

```verilog
//执行级计算得到访存地址，需要传回写回级供页表相关处理
assign error_va        = pv_addr;
```
执行级的会产生的例外相关信号如下，这一级会得到访存地址，将判断ale例外
```verilog
//exception
assign excp_ale        = access_mem & ((es_mem_size[0] &  1'b0)                                  | 
                                       (es_mem_size[1] &  es_alu_result[0])                      | 
                                       (!es_mem_size   & (es_alu_result[0] | es_alu_result[1]))) ;
assign excp            = es_excp || excp_ale;
assign excp_num        = {excp_ale, es_excp_num};
```

```verilog
//sram
//生成访存位掩码信号，用于支持.b .h类型的st指令
assign sram_addr_low2bit = {es_alu_result[1], es_alu_result[0]};
//mem_size[0] byte size   [1] halfword size
//dcache_req_or_inst_en反映了当前流水级是否应该产生效果和向后流水
assign dcache_req_or_inst_en = es_valid && !excp && ms_allowin && !es_flush_sign && !ms_flush;
//data_valid不止要考虑当前是不是访存指令，也要考虑当前流水线有没有特殊情况（即dcache_req_or_inst_en不有效），以防止发出不应该发出的访存命令
//类似的“做了不该做”的事情，是debug过程中一个比较常见而比较难发现的错误
//因此，任何对当前流水线产生影响的事情，都要仔细考虑“应不应该发出”（比如要考虑当前流水级的valid是否有效等）
assign data_valid = access_mem && dcache_req_or_inst_en;
//data_op为高代表对dcache的操作是写类型
//这里暂时不理解为什么要有后面亮两项，cacop和preld指令时，store_op都不为高
assign data_op    = es_store_op && !es_cacop && !es_preld;
assign data_wstrb = wr_byte_en;
//特例：tlbsrch指令时，访存地址来自于csr_vppn
assign data_addr = es_tlbsrch ? {csr_vppn, 13'b0} : pv_addr;
//hex/bit类型的存储指令的写掩码逻辑，建议这里学习这种判断逻辑———》独热码——》独热码多选器的写法
wire [3:0] es_stb_wen = { sram_addr_low2bit==2'b11  ,
                          sram_addr_low2bit==2'b10  ,
                          sram_addr_low2bit==2'b01  ,
                          sram_addr_low2bit==2'b00} ;
wire [3:0] es_sth_wen = { sram_addr_low2bit==2'b10  ,
                          sram_addr_low2bit==2'b10  ,
                          sram_addr_low2bit==2'b00  ,
                          sram_addr_low2bit==2'b00} ;
wire [31:0] es_stb_cont = { {8{es_stb_wen[3]}} & es_rkd_value[7:0] ,
                            {8{es_stb_wen[2]}} & es_rkd_value[7:0] ,
                            {8{es_stb_wen[1]}} & es_rkd_value[7:0] ,
                            {8{es_stb_wen[0]}} & es_rkd_value[7:0]};
wire [31:0] es_sth_cont = { {16{es_sth_wen[3]}} & es_rkd_value[15:0] ,
                            {16{es_sth_wen[0]}} & es_rkd_value[15:0]};
                        
//建议学习这种写法，判断条件相同而共同产生功能（这里说的共同产生功能，说的是这两个信号实际上都是在访存指令位数不同的情况下，产生的一组用于访存接口的信号）时用拼接，同时写两个信号的逻辑，逻辑更清晰且代码更简洁，
//在访存指令位数不同的情况下，产生的一组用于访存接口的信号
assign {wr_byte_en, data_size}  = ({7{es_mem_size[0]}} & {es_stb_wen, 3'b00}) |
                                  ({7{es_mem_size[1]}} & {es_sth_wen, 3'b01}) |
                                  ({7{!es_mem_size  }} & {4'b1111   , 3'b10}) ;                                  
//assign data_wdata = es_rkd_value; 
//存储到dcache的数据
assign data_wdata = ({32{es_mem_size[0]}} & es_stb_cont ) |
                    ({32{es_mem_size[1]}} & es_sth_cont ) |
                    ({32{!es_mem_size  }} & es_rkd_value) ;
```

与页表相关的信号逻辑生成如下
与页表相关的硬件实现，仔细阅读和理解《设计实战》中页表部分的注意事项和参考la32r指令集中页表的相关描述，硬件方面的工作就可以比较轻松的完成
个人认为，关于页表的相关实现，其实重在“理解页表可以做什么工作（配合软件的视角去看）”，连带着理解CSR和特权态相关内容，这里其实是可以学出差异化的，花点时间，去理解“加入CSR、特权态、页表，为我的CPU带来了什么”，理解的好在个人看来是相当加分的，毕竟这本质上是CPU要能“用的起来”的重要硬件部分（笔者面试就被问到过页表的规格问题）
顺带一提，读者如果学有余力，推荐完成操作系统的经典实验xv6 lab（mit 6.s081），对页表和CPU的理解也会更上一层楼
```verilog
//注意这里存在页表的数据冒险，因为一般认为页表操作发生频率不高，所以采用stall的方式解决比较科学
assign tlbsrch_stall = es_tlbsrch && ms_wr_tlbehi;
//invtlb 
assign es_invtlb_asid = es_rj_value[9:0];
assign es_invtlb_vpn  = es_rkd_value[31:13];
assign pv_addr = es_alu_result;
```

```verilog
assign data_fetch = (data_valid || dcacop_inst || preld_en) && data_addr_ok || ((icacop_inst || es_tlbsrch) && es_ready_go && ms_allowin);
```
i/d cache相关信号如下，执行级除了普通访存外，与cache的互动逻辑中，还有与cacop指令相关的逻辑信号
```verilog
//cache ins
//按照定义实现的一系列cacop指令逻辑
assign cacop_op         = es_dest;
assign icacop_inst      = es_cacop && (cacop_op[2:0] == 3'b0);
assign icacop_op_en     = icacop_inst && dcache_req_or_inst_en;
assign dcacop_inst      = es_cacop && (cacop_op[2:0] == 3'b1);
assign dcacop_op_en     = dcacop_inst && dcache_req_or_inst_en;
assign cacop_op_mode    = cacop_op[4:3];
```
preld指令相关信号实现如下
```verilog
//preld ins
assign preld_hint = es_dest;
assign preld_inst = es_preld && ((preld_hint == 5'd0) || (preld_hint == 5'd8))/* && !data_uncache_en*/; //preld must have bug
assign preld_en   = preld_inst && dcache_req_or_inst_en;

```

