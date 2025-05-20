
跳转总线如下所示。本设计中，跳转结果直接在译码级结果计算出来，不用到执行级再做，这能减少流水线的预测错误开销，并且方便好写。
```verilog
						//本设计中，跳转结果在译码级结果计算出来、
						//传回的跳转总线提供冲刷信息，即冲刷有效信号和目标地址
assign br_bus       = {btb_pre_error_flush,           //32:32
                       btb_pre_error_flush_target     //31:0
                      };
```
译码往执行级的总线信号中，包括的必要信息如下所示
```verilog
						//difftest使用信号
assign ds_to_es_bus = {inst_csr_rstat_en,  // 349:349 for difftest
                       inst_st_en       ,  // 348:341 for difftest
                       inst_ld_en       ,  // 340:333 for difftest
                       (inst_rdcntvl_w | inst_rdcntvh_w | inst_rdcntid_w), //332:332  for difftest
                       timer_64      ,  //331:268  for difftest
                       ds_inst       ,  //267:236  for difftest
						//指令为idle指令
                       inst_idle     ,  //235:235
						//下四个信号均为性能计数器用
                       btb_pre_error_flush, //234:234
                       br_to_btb     ,  //233:233
                       ds_icache_miss,  //232:232
                       br_inst       ,  //231:231
						//preld是la32r定义的icache预取指令，需特殊标记
                       inst_preld    ,  //230:230
	                    //表明是cacop指令且有效（能作用在本处理器核，如操作L2cache就属于无效）
                       inst_valid_cacop,  //229:229
	                    //表明当前是需对load结果进行符号拓展的load信号（即ld.b和ld.h),用于mem级生成ld指令的写回寄存器结果
                       mem_sign_exted,  //228:228
						//页表无效指令，需标记起来一直传到写回级操作页表
                       inst_invtlb   ,  //227:227
						//页表读取指令，需标记起来一直传到写回级操作页表
                       inst_tlbrd    ,  //226:226
						//与tlb/ibar指令相关的refetch指令，一直传回到写回级生成refetch信号
                       refetch       ,  //225:225
						//页表填充指令，需标记起来一直传到写回级操作页表
                       inst_tlbfill  ,  //224:224
						//页表写指令，需标记起来一直传到写回级操作页表
                       inst_tlbwr    ,  //223:223
						//页表搜寻指令，需标记起来一直传到写回级操作页表
                       inst_tlbsrch  ,  //222:222
						//一对原子访存指令，需标记至访存级
                       inst_sc_w     ,  //221:221
                       inst_ll_w     ,  //220:220
						//例外信号，这里相比于取指到译码的总线，因可能的例外随着流水级增高而增多，位宽增加
                       excp_num      ,  //219:211
						//csr相关信号，写csr相关放到写回去做，所以要一直传下去
                       csr_mask      ,  //210:210
                       csr_we        ,  //209:209
                       csr_idx       ,  //208:195
                       res_from_csr  ,  //194:194
                       csr_data      ,  //193:162
						//表明是ertn指令，需要标记起来传递到写回级，供写回级发出冲刷信号
                       inst_ertn     ,  //161:161
						//表明发生例外，需要标记起来传递到写回级，供写回级发出冲刷信号
                       excp          ,  //160:160
						//标记访存指令访存字节数的信号，需要传递到访存级
                       mem_size      ,  //159:158
						//将用以区分各种乘/除指令的详细信息传递
                       mul_div_op    ,  //157:154
                       mul_div_sign  ,  //153:153
						//将用以区分各种alu指令的详细信息传递
                       alu_op        ,  //152:139
						//表明当前指令是加载指令
                       load_op       ,  //138:138 bug2 load_op
						//一系列表明源操作数来源的信号，源操作数的生成是在执行级做的
                       src1_is_pc    ,  //137:137
                       src2_is_imm   ,  //136:136
                       src2_is_4     ,  //135:135
						//写寄存器使能信号，一直传到写回级
                       gr_we         ,  //134:134
						//表明当前指令是存储指令
                       store_op      ,  //133:133
						//目标写寄存器，一直传到写回级
                       dest          ,  //132:128
						//可能的四个源操作数来源，传递到执行级：立即数，两个寄存器值，pc的值
                       ds_imm        ,  //127:96
                       rj_value      ,  //95 :64
                       rkd_value     ,  //63 :32
                       ds_pc            //31 :0
                      };
```
译码流水线的控制信号解读如下，stall的发生除了屏障和页表操作之外，就是和数据冲突冒险有关，所以还算比较清晰
```verilog
//译码级的stall主要是三种情况:1.无法通过前递解决的数据冒险冲突 2.后三级有tlb指令需要暂停 3.直到后面的流水线清空，才能继续执行屏障指令 
//发生例外时，代表当前指令无效，所以无条件go
assign ds_ready_go    = !(rf2_forward_stall || rf1_forward_stall/*|| idle_stall*/ || tlb_inst_stall || ibar_stall || dbar_stall) || excp;
//标准的allow写法
assign ds_allowin     = !ds_valid || ds_ready_go && es_allowin;
//标准的valid写法
assign ds_to_es_valid = ds_valid && ds_ready_go;
always @(posedge clk) begin   //bug1 no reset; branch no delay slot
    if (reset || flush_sign) begin
		//流水线冲刷无条件置低valid
        ds_valid <= 1'b0;
    end
    else begin 
        if (ds_allowin) begin   //bug2 ??
			//当该分支指令向后流动的那一拍，即（btb_pre_error_flush && es_allowin）为1时，若下一拍下一条指令紧接着进入译码级，则可直接取消。
			//不过可以用更粗犷的方法，即下一拍的ds_valid直接为0，无需识别是否有指令进入
			//同样的条件，若下一拍无指令进入，即当拍fs_to_ds_valid为0，则置起branch_slot_cancel触发器。
			//该触发器为1时，会等待下一条指令进入，当检测fs_to_ds_valid为1时，将其取消，并恢复branch_slot_cancel至0。
            if ((btb_pre_error_flush && es_allowin) || branch_slot_cancel) begin
                ds_valid <= 1'b0;
            end
            else begin
			    //标准写法
                ds_valid <= fs_to_ds_valid;
            end
        end
    end
```
以下是流水线寄存器的寄存逻辑写法，由于采用了“打包”的方法，所以寄存逻辑只需要一行就写完了（读者可以自行想象，不用打包成总线的写法，会多出多少工作量）。如《设计实战》中提及的，流水线寄存器中存储的内容都是与它输出那一级流水的组合逻辑相关联的，所以把fs/ds流水线寄存器放到译码一级比较合适，理解起来也更直观
```verilog
    //流水线寄存器标准写法
    //如《设计实战》中提及的，流水线寄存器中存储的内容都是与它输出那一级流水的组合逻辑相关联的，所以把fs/ds流水线寄存器放到译码一级比较合适
    if (fs_to_ds_valid && ds_allowin) begin
        fs_to_ds_bus_r <= fs_to_ds_bus;
    end
end
```
以下是对流水线寄存器“打包”的信号重新“解开”以供使用的标准写法
>在stageFe解读中，展示了流水线级间总线的标准写法，这种“打包信号”的写法可以省去很多工作量（如顶层例化时简便很多，增减流水线间的信号也很方便）
```verilog
assign {ds_btb_target,  //108:77
        ds_btb_index,   //76:72
        ds_btb_taken,   //71:71
        ds_btb_en,      //70:70
        ds_icache_miss, //69:69
        ds_excp,        //68:68
        ds_excp_num,    //67:64
        ds_inst,        //63:32
        ds_pc           //31:0
       } = fs_to_ds_bus_r;
```
指令译码的逻辑中，操作码摘取和指令翻译出来的逻辑如下所示：
```verilog
//按照指令集手册，分别摘取指令中的操作码赋成变量，方便进行译码
//操作类型字段
assign op_31_26  = ds_inst[31:26];
……
//寄存器字段
assign rd   = ds_inst[ 4: 0];
……
//立即数字段
assign i12  = ds_inst[21:10];
……
//csr相关的字段
assign csr_idx = ds_inst[23:10];
//手动用解码器将操作码译成独热码，避免重复编写解码逻辑，也可以让整个译码逻辑综合出来的电路性能更好
decoder_6_64 u_dec0(.in(op_31_26 ), .out(op_31_26_d ));
decoder_4_16 u_dec1(.in(op_25_22 ), .out(op_25_22_d ));
decoder_2_4  u_dec2(.in(op_21_20 ), .out(op_21_20_d ));
decoder_5_32 u_dec3(.in(op_19_15 ), .out(op_19_15_d ));
decoder_5_32 u_dec4(.in(rd  ), .out(rd_d  ));
decoder_5_32 u_dec5(.in(rj  ), .out(rj_d  ));
decoder_5_32 u_dec6(.in(rk  ), .out(rk_d  ));
//采用直译的形式将指令类型直接翻译出来，简洁易懂方便维护，eda工具一般也会将冗余的逻辑优化，不用太担心性能
assign inst_add_w      = op_31_26_d[6'h00] & op_25_22_d[4'h0] & op_21_20_d[2'h1] & op_19_15_d[5'h00];
assign inst_sub_w      = op_31_26_d[6'h00] & op_25_22_d[4'h0] & op_21_20_d[2'h1] & op_19_15_d[5'h02];
……
```
得到翻译出来的指令具体类型后，根据具体类型将各种需要传递下去的信息（alu操作类型、源操作数来源、访存的位数等等）表示出来，逻辑比较易懂，这里不全部列举出来
```verilog
//翻译指令的alu操作类型
assign alu_op[ 0] = inst_add_w      | 
                    inst_addi_w     | 
                    inst_ld_b       |
                    inst_ld_h       |
                    inst_ld_w       |
                    inst_st_b       |
                    inst_st_h       | 
                    inst_st_w       |
                    inst_ld_bu      |
                    inst_ld_hu      | 
                    inst_ll_w       |
                    inst_sc_w       |
                    inst_jirl       | 
                    inst_bl         |
                    inst_pcaddi     |
                    inst_pcaddu12i  |
                    inst_valid_cacop|
                    inst_preld      ;
assign alu_op[ 1] = inst_sub_w;
……
```
openLA500的实现中，将寄存器堆和与之对应的交互逻辑（还包括写回级的写回寄存器总线，都是输入到译码级的）都放在译码级做了，对于经典五级流水线+异步读寄存器堆来说，这种做法应该是最简单方便的
```verilog
//写回级写回寄存器堆的总线解包，包含了写使能、写地址、写数据三条信息
assign {rf_we   ,  //37:37
        rf_waddr,  //36:32
        rf_wdata   //31:0
       } = ws_to_rf_bus;
//infor_flag是方便debug用的信号
//以下两行是读寄存器信号的生成
assign rf_raddr1 = infor_flag?reg_num:rj;
assign rf_raddr2 = src_reg_is_rd ? rd : rk;
//例化寄存器堆模块与译码级内
regfile u_regfile(
    .clk    (clk      ),
    .raddr1 (rf_raddr1),
    .rdata1 (rf_rdata1),
    .raddr2 (rf_raddr2),
    .rdata2 (rf_rdata2),
    .we     (rf_we    ),
    .waddr  (rf_waddr ),
    .wdata  (rf_wdata )
//difftest用信号
    `ifdef DIFFTEST_EN
    ,
    .rf_o   (rf_to_diff)
    `endif
    );
```
译码级中，解决数据冒险冲突的逻辑实现如下所示
>有流水线级的CPU实现中，一大错误来源就是数据冒险冲突没有处理好，读者自行实现时，发现写回的计算结果错误而debug的时候不妨先看看源操作数是不是都对了，如果有错误，就可以看看是不是数据冲突问题（这里说的是比较广义的，不止是寄存器冲突，还包括流水线没把应该传递的结果在**那一拍**传递到等）

```verilog
	  //注意各个数据冲突总线，除了包含使能、寄存器号、写的数据三个写寄存器的内容外，还包括了“需要暂停”这一信息
	  //执行级无法在执行级当拍得到写寄存器的结果的时候，如果产生冲突就会需要停顿，在本实现中，为：乘法操作，除法操作和load操作
assign {es_dep_need_stall,
        es_forward_enable, 
        es_forward_reg   ,
        es_forward_data
       } = es_to_ds_forward_bus;
       //访存级主要是无法在到达访存级当拍得到访存结果的时候（dcache缺失），如果产生冲突就会需要停顿
assign {ms_dep_need_stall,
        ms_forward_enable, 
        ms_forward_reg   ,
        ms_forward_data
       } = ms_to_ds_forward_bus;
//exe stage first forward
		//“打包”的赋值语句写法，比较简洁
		//注意冲突有效还要注意该条指令是不是真的需要读寄存器值，即inst_need_r*
		//
assign {rf1_forward_stall, rj_value, rj_value_forward_es} = ((rf_raddr1 == es_forward_reg) && es_forward_enable && inst_need_rj) ? {es_dep_need_stall, es_forward_data, es_forward_data} :
                                                            ((rf_raddr1 == ms_forward_reg) && ms_forward_enable && inst_need_rj) ? {ms_dep_need_stall || br_need_reg_data, ms_forward_data, rf_rdata1} :
                                                                                                                                   {1'b0, rf_rdata1, rf_rdata1}; 
assign {rf2_forward_stall, rkd_value, rkd_value_forward_es} = ((rf_raddr2 == es_forward_reg) && es_forward_enable && inst_need_rkd) ? {es_dep_need_stall, es_forward_data, es_forward_data} :
                                                              ((rf_raddr2 == ms_forward_reg) && ms_forward_enable && inst_need_rkd) ? {ms_dep_need_stall || br_need_reg_data, ms_forward_data, rf_rdata2} :
                                                                                                                                      {1'b0, rf_rdata2, rf_rdata2};
```


