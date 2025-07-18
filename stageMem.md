访存阶段是五级流水线中的第四级，主要负责处理内存访问操作、页表转换、异常检测等关键功能。该阶段需要处理从执行阶段传来的访存请求，并进行地址转换和权限检查。

`include "mycpu.h"
`include "csr.h"

module mem_stage(
    input                              clk           ,
    input                              reset         ,
    //allowin
    input                              ws_allowin    ,
    output                             ms_allowin    ,
    //from es
    input                              es_to_ms_valid,
    input  [`ES_TO_MS_BUS_WD -1:0]     es_to_ms_bus  ,
    //to ws
    output                             ms_to_ws_valid,
    output [`MS_TO_WS_BUS_WD -1:0]     ms_to_ws_bus  ,
    //to ds forward path 
    output [`MS_TO_DS_FORWARD_BUS-1:0] ms_to_ds_forward_bus,
    output                             ms_to_ds_valid,
    //div mul
    input  [31:0]     div_result    ,
    input  [31:0]     mod_result    ,
    input  [63:0]     mul_result    ,
    //exception
    input             excp_flush    ,
    input             ertn_flush    ,
    input             refetch_flush ,
    input             icacop_flush  ,
    //idle
    input             idle_flush    ,
    //tlb ins
    output            tlb_inst_stall,
    //to es 
    output            ms_wr_tlbehi  ,
    output            ms_flush      ,
    //from cache
    input             data_data_ok   ,
    input             dcache_miss    ,
    input  [31:0]     data_rdata     ,
    //to cache
    output            data_uncache_en,
    output            tlb_excp_cancel_req,
    output            sc_cancel_req  ,
    //from csr 
    input             csr_pg         ,
    input             csr_da         ,
    input  [31:0]     csr_dmw0       ,
    input  [31:0]     csr_dmw1       ,
    input  [ 1:0]     csr_plv        ,
    input  [ 1:0]     csr_datm       ,
    input             disable_cache  ,
    input  [27:0]     lladdr         ,
    // from addr trans for difftest
    input  [ 7:0]     data_index_diff   ,
    input  [19:0]     data_tag_diff     ,
    input  [ 3:0]     data_offset_diff  ,
    //to addr trans 
    output            data_addr_trans_en,   
    output            dmw0_en           ,
    output            dmw1_en           ,
    output            cacop_op_mode_di  ,   
    //tlb 
    input             data_tlb_found ,
    input  [ 4:0]     data_tlb_index ,
    input             data_tlb_v     ,
    input             data_tlb_d     ,
    input  [ 1:0]     data_tlb_mat   ,
    input  [ 1:0]     data_tlb_plv   ,
    input  [19:0]     data_tlb_ppn   
);

// 从执行级传来的信号解析，访存级的主要输入总线
reg         ms_valid;
wire        ms_ready_go;

wire        dep_need_stall;
reg [`ES_TO_MS_BUS_WD -1:0] es_to_ms_bus_r;
wire [ 3:0] ms_mul_div_op;
wire [ 1:0] sram_addr_low2bit;
wire [ 1:0] ms_mem_size;
wire        ms_load_op;
wire        ms_gr_we;
wire [ 4:0] ms_dest;
wire [31:0] ms_exe_result;
wire [31:0] ms_pc;
wire        ms_excp;
wire [ 9:0] ms_excp_num;
wire        ms_ertn;
wire [31:0] ms_csr_result;
wire [13:0] ms_csr_idx;
wire        ms_csr_we;
wire        ms_ll_w;
wire        ms_sc_w;
wire        ms_store_op;
wire        ms_tlbsrch;
wire        ms_tlbfill;
wire        ms_tlbwr;
wire        ms_tlbrd;
wire        ms_refetch;
wire        ms_invtlb;
wire [ 9:0] ms_invtlb_asid;
wire [18:0] ms_invtlb_vpn;
wire        ms_mem_sign_exted;
wire        ms_icacop_op_en;
wire        ms_br_inst;
wire        ms_icache_miss;
wire        ms_br_pre;
wire        ms_br_pre_error;
wire        ms_preld_inst;
wire        ms_cacop;
wire        ms_idle;
wire [31:0] ms_error_va;

// difftest
wire        ms_cnt_inst     ;
wire [63:0] ms_timer_64     ;
wire [31:0] ms_inst         ;
wire [ 7:0] ms_inst_ld_en   ;
wire [31:0] ms_ld_paddr     ;
wire [31:0] ms_ld_vaddr     ;
wire [ 7:0] ms_inst_st_en   ;
wire [31:0] ms_st_data      ;
wire        ms_csr_rstat_en ;
wire [31:0] ms_csr_data     ;

//从执行级传来的总线信号解包，包含访存级需要处理的所有信息
assign {ms_csr_data      ,  //424:393  for difftest
        ms_csr_rstat_en  ,  //392:392  for difftest
        ms_st_data       ,  //391:360  for difftest
        ms_inst_st_en    ,  //359:352  for difftest
        ms_ld_vaddr      ,  //351:320  for difftest
        ms_inst_ld_en    ,  //319:312  for difftest
        ms_cnt_inst      ,  //311:311  for difftest
        ms_timer_64      ,  //310:247  for difftest
        ms_inst          ,  //246:215  for difftest
        ms_error_va      ,  //214:183
        ms_idle          ,  //182:182
        ms_cacop         ,  //181:181
        ms_preld_inst    ,  //180:180
        ms_br_pre_error  ,  //179:179
        ms_br_pre        ,  //178:178
        ms_icache_miss   ,  //177:177
        ms_br_inst       ,  //176:176
        ms_icacop_op_en  ,  //175:175
        ms_mem_sign_exted,  //174:174
        ms_invtlb_vpn    ,  //173:155
        ms_invtlb_asid   ,  //154:145
        ms_invtlb        ,  //144:144
        ms_tlbrd         ,  //143:143
        ms_refetch       ,  //142:142
        ms_tlbfill       ,  //141:141
        ms_tlbwr         ,  //140:140
        ms_tlbsrch       ,  //139:139
        ms_store_op      ,  //138:138
        ms_sc_w          ,  //137:137
        ms_ll_w          ,  //136:136
        ms_excp_num      ,  //135:126
        ms_csr_we        ,  //125:125
        ms_csr_idx       ,  //124:111
        ms_csr_result    ,  //110:79
        ms_ertn          ,  //78:78
        ms_excp          ,  //77:77
        ms_mem_size      ,  //76:75
        ms_mul_div_op    ,  //74:71
        ms_load_op       ,  //70:70
        ms_gr_we         ,  //69:69
        ms_dest          ,  //68:64
        ms_exe_result    ,  //63:32
        ms_pc               //31:0
       } = es_to_ms_bus_r;

//访存级内部信号定义
wire [31:0] mem_result;        //访存指令的读取结果
wire [31:0] ms_final_result;   //访存级最终写回的结果
wire        flush_sign;        //冲刷信号

wire [31:0] ms_rdata;          //实际使用的读数据（考虑缓存）
reg  [31:0] data_rd_buff;      //数据缓存寄存器
reg         data_buff_enable;  //数据缓存使能

wire        access_mem;        //是否需要访问内存

wire [ 4:0] cacop_op;          //Cache操作码
wire [ 1:0] cacop_op_mode;     //Cache操作模式

wire        forward_enable;    //前递使能信号
wire        dest_zero;         //目标寄存器为0

wire [31:0] paddr;             //物理地址

wire [15:0] excp_num;          //例外编号
wire        excp;              //例外发生标志

//页表相关例外信号
wire        excp_tlbr;         //TLB重填例外
wire        excp_pil ;         //页面无效例外（load）
wire        excp_pis ;         //页面无效例外（store）
wire        excp_pme ;         //页面修改例外
wire        excp_ppi ;         //页面特权等级例外

//地址转换模式
wire        da_mode  ;         //直接地址模式
wire        pg_mode  ;         //分页模式

wire        sc_addr_eq;        //SC指令地址比较结果

//访存级往写回级传递的信号总线，包含所有需要传递的信息
//总线位宽达到493位，承载了大量的控制信息和数据
assign ms_to_ws_bus = {ms_csr_data    ,  //492:461 for difftest - CSR数据，用于验证
                       ms_csr_rstat_en,  //460:460 for difftest - CSR状态寄存器读使能
                       ms_st_data     ,  //459:428 for difftest - 存储数据，用于验证
                       ms_inst_st_en  ,  //427:420 for difftest - 存储指令使能向量
                       ms_ld_vaddr    ,  //419:388 for difftest - 加载虚拟地址
                       ms_ld_paddr    ,  //387:356 for difftest - 加载物理地址
                       ms_inst_ld_en  ,  //355:348 for difftest - 加载指令使能向量
                       ms_cnt_inst    ,  //347:347 for difftest - 计数器指令标志
                       ms_timer_64    ,  //346:283 for difftest - 64位计时器值
                       ms_inst        ,  //282:251 for difftest - 指令码
					   data_uncache_en,  //250:250 非缓存访问使能，指示是否绕过Cache
					   paddr          ,  //249:218 物理地址（32位），TLB转换后的最终访问地址
                       ms_idle        ,  //217:217 IDLE指令标志，处理器进入低功耗模式
                       ms_br_pre_error,  //216:216 分支预测错误标志，用于性能统计
                       ms_br_pre      ,  //215:215 分支预测标志，表示是否进行了分支预测
                       dcache_miss    ,  //214:214 数据Cache未命中标志，用于性能统计
                       access_mem     ,  //213:213 访存操作标志，表示当前是否为访存指令
                       ms_icache_miss ,  //212:212 指令Cache未命中标志，从前级传递
                       ms_br_inst     ,  //211:211 分支指令标志，用于BTB更新和性能统计
                       ms_icacop_op_en,  //210:210 指令Cache操作使能，需要在写回级处理
                       ms_invtlb_vpn  ,  //209:191 INVTLB指令虚拟页号（19位），用于TLB失效操作
                       ms_invtlb_asid ,  //190:181 INVTLB指令地址空间ID（10位）
                       ms_invtlb      ,  //180:180 INVTLB指令标志，TLB表项失效指令
                       ms_tlbrd       ,  //179:179 TLBRD指令标志，TLB读取指令
                       ms_refetch     ,  //178:178 重取指标志，某些指令执行后需要重新取指
                       ms_tlbfill     ,  //177:177 TLBFILL指令标志，TLB填充指令
                       ms_tlbwr       ,  //176:176 TLBWR指令标志，TLB写入指令
                       data_tlb_index ,  //175:171 TLB表项索引（5位），标识命中的TLB表项
                       data_tlb_found ,  //170:170 TLB查找命中标志，表示在TLB中找到对应表项
                       ms_tlbsrch     ,  //169:169 TLBSRCH指令标志，TLB搜索指令
                       ms_error_va    ,  //168:137 错误虚拟地址（32位），用于例外处理和页表操作
                       ms_sc_w        ,  //136:136 SC（Store Conditional）指令标志
                       ms_ll_w        ,  //135:135 LL（Load Linked）指令标志
                       excp_num       ,  //134:119 例外类型编号（16位），包含所有可能的例外类型
                       ms_csr_we      ,  //118:118 CSR写使能，控制CSR寄存器的写入操作
                       ms_csr_idx     ,  //117:104 CSR寄存器索引（14位），指定要操作的CSR寄存器
                       ms_csr_result  ,  //103:72  CSR操作结果（32位），写入CSR的数据
                       ms_ertn        ,  //71:71   ERTN指令标志，例外返回指令
                       excp           ,  //70:70   例外发生总标志，任何例外发生时置1
                       ms_gr_we       ,  //69:69   通用寄存器写使能，控制是否写回寄存器
                       ms_dest        ,  //68:64   目标寄存器号（5位），指定写回的寄存器
                       ms_final_result,  //63:32   最终写回结果（32位），可能来自访存、乘除法等
                       ms_pc             //31:0    当前指令PC值（32位）
                      };

//标准流水线控制信号
assign ms_to_ds_valid = ms_valid;

//访存级ready信号逻辑：需要等待数据cache响应或不需要访存时ready
//当例外发生或SC指令取消时，无条件ready以简化控制逻辑
assign ms_ready_go    = (data_data_ok || data_buff_enable) || !access_mem || excp || sc_cancel_req;
assign ms_allowin     = !ms_valid || ms_ready_go && ws_allowin;
assign ms_to_ws_valid = ms_valid && ms_ready_go;
//标准流水线valid信号写法
always @(posedge clk) begin
    if (reset || flush_sign) begin
        ms_valid <= 1'b0;
    end
    else if (ms_allowin) begin
        ms_valid <= es_to_ms_valid;
    end

    if (es_to_ms_valid && ms_allowin) begin
        es_to_ms_bus_r <= es_to_ms_bus;
    end
end                            

//判定是否需要访问内存
assign access_mem = ms_store_op || ms_load_op;

//冲刷信号汇总，来自各种异常和特殊指令
assign flush_sign = excp_flush || ertn_flush || refetch_flush || icacop_flush || idle_flush;

//实际使用的读数据：优先使用缓存的数据，否则使用当前周期的数据
assign ms_rdata = data_buff_enable ? data_rd_buff : data_rdata;

//访存地址的低2位，用于字节/半字访问的数据选择
assign sram_addr_low2bit = {ms_exe_result[1], ms_exe_result[0]};

//字节访问数据选择逻辑，根据地址低2位选择对应字节
wire [7:0] mem_byteLoaded = ({8{sram_addr_low2bit==2'b00}} & ms_rdata[ 7: 0]) |
                            ({8{sram_addr_low2bit==2'b01}} & ms_rdata[15: 8]) |
                            ({8{sram_addr_low2bit==2'b10}} & ms_rdata[23:16]) |
                            ({8{sram_addr_low2bit==2'b11}} & ms_rdata[31:24]) ; 
                                                            

//半字访问数据选择逻辑，根据地址低2位选择对应半字
wire [15:0] mem_halfLoaded = ({16{sram_addr_low2bit==2'b00}} & ms_rdata[15: 0]) |
                             ({16{sram_addr_low2bit==2'b10}} & ms_rdata[31:16]) ;

//访存结果生成，根据访存大小和符号扩展进行数据处理
//支持字节(.b)、半字(.h)、字(.w)三种访存大小，以及有符号/无符号扩展
assign mem_result = ({32{ms_mem_size[0] &&  ms_mem_sign_exted}} & {{24{mem_byteLoaded[ 7]}}, mem_byteLoaded}) |
                    ({32{ms_mem_size[0] && ~ms_mem_sign_exted}} & { 24'b0                  , mem_byteLoaded}) |
                    ({32{ms_mem_size[1] &&  ms_mem_sign_exted}} & {{16{mem_halfLoaded[15]}}, mem_halfLoaded}) |
                    ({32{ms_mem_size[1] && ~ms_mem_sign_exted}} & { 16'b0                  , mem_halfLoaded}) |
                    ({32{!ms_mem_size}}                         &   ms_rdata                                  ) ;

//最终写回结果选择：根据指令类型选择相应的结果
//包括访存结果、乘除法结果、执行级计算结果等
assign ms_final_result = ({32{ms_load_op      }} & mem_result       )  |
                         ({32{ms_mul_div_op[0]}} & mul_result[31:0] )  |
                         ({32{ms_mul_div_op[1]}} & mul_result[63:32])  |
                         ({32{ms_mul_div_op[2]}} & div_result       )  |
                         ({32{ms_mul_div_op[3]}} & mod_result       )  |
                         ({32{!ms_mul_div_op && !ms_load_op}} & (ms_exe_result&{32{!sc_cancel_req}}));

//前递通路逻辑，标准写法，用于解决数据冒险
assign dest_zero            = (ms_dest == 5'b0);
assign forward_enable       = ms_gr_we & ~dest_zero & ms_valid;
//访存指令且尚未完成时需要stall
assign dep_need_stall       = ms_load_op && !ms_to_ws_valid;
assign ms_to_ds_forward_bus = {dep_need_stall,  //38:38
                               forward_enable,  //37:37
                               ms_dest       ,  //36:32
                               ms_final_result  //31:0
                              };

//地址转换相关逻辑
//分页模式：开启页表功能且关闭直接地址模式
assign pg_mode = !csr_da && csr_pg;
//直接地址模式：开启直接地址且关闭页表功能
assign da_mode =  csr_da && !csr_pg;

//需要进行TLB地址转换的条件：分页模式下且不使用DMW窗口且不是DI类型的CACOP指令
assign data_addr_trans_en = pg_mode && !dmw0_en && !dmw1_en && !cacop_op_mode_di;

//TLB转换后的物理地址
assign paddr = {data_tlb_ppn, ms_error_va[11:0]};

//SC指令地址匹配检查：比较LL指令记录的地址与当前访问地址
assign sc_addr_eq = (lladdr == paddr[31:4]);
//SC指令取消条件：地址不匹配或访问非缓存区域
assign sc_cancel_req = (!sc_addr_eq||data_uncache_en) && ms_sc_w && access_mem;

//DMW（直接映射窗口）使能逻辑，用于高速地址转换
//DMW0窗口使能：匹配特权级和虚拟地址段
assign dmw0_en = ((csr_dmw0[`PLV0] && csr_plv == 2'd0) || (csr_dmw0[`PLV3] && csr_plv == 2'd3)) && (ms_error_va[31:29] == csr_dmw0[`VSEG]) && pg_mode;
//DMW1窗口使能：匹配特权级和虚拟地址段  
assign dmw1_en = ((csr_dmw1[`PLV0] && csr_plv == 2'd0) || (csr_dmw1[`PLV3] && csr_plv == 2'd3)) && (ms_error_va[31:29] == csr_dmw1[`VSEG]) && pg_mode;

//例外信号汇总
assign excp = excp_tlbr || excp_pil || excp_pis || excp_ppi || excp_pme || ms_excp;
assign excp_num = {excp_pil, excp_pis, excp_ppi, excp_pme, excp_tlbr, 1'b0, ms_excp_num};

//TLB相关例外判断，这些例外只在特定访存操作中产生
//注意：PRELD指令不应产生这些例外
assign excp_tlbr = (access_mem || ms_cacop) && !data_tlb_found && data_addr_trans_en;      //TLB重填例外
assign excp_pil  = (ms_load_op || ms_cacop) && !data_tlb_v && data_addr_trans_en;         //页无效例外（load）
assign excp_pis  = ms_store_op && !data_tlb_v && data_addr_trans_en;                      //页无效例外（store）
assign excp_ppi  = access_mem && data_tlb_v && (csr_plv > data_tlb_plv) && data_addr_trans_en;  //页特权等级例外
assign excp_pme  = ms_store_op && data_tlb_v && (csr_plv <= data_tlb_plv) && !data_tlb_d && data_addr_trans_en;  //页修改例外

//TLB例外时取消cache请求
assign tlb_excp_cancel_req = excp_tlbr || excp_pil || excp_pis || excp_ppi || excp_pme;

//非缓存访问判断逻辑，多种情况下需要绕过cache直接访问内存
assign data_uncache_en = (da_mode && (csr_datm == 2'b0))                 ||   //直接地址模式且MAT为0
                         (dmw0_en && (csr_dmw0[`DMW_MAT] == 2'b0))       ||   //DMW0窗口且MAT为0
                         (dmw1_en && (csr_dmw1[`DMW_MAT] == 2'b0))       ||   //DMW1窗口且MAT为0
                         (data_addr_trans_en && (data_tlb_mat == 2'b0))  ||   //TLB转换且MAT为0
                         disable_cache;                                        //强制禁用cache

//访存级冲刷信号：各种会改变处理器状态的指令和例外
assign ms_flush = (excp | ms_ertn | (ms_csr_we | (ms_ll_w | ms_sc_w) & !excp) | ms_refetch | ms_idle) & ms_valid;

//TLB指令停顿：TLBSRCH和TLBRD指令需要停顿流水线
assign tlb_inst_stall = (ms_tlbsrch || ms_tlbrd) && ms_valid;

//数据缓存逻辑：当写回级无法接收时，暂存数据cache的返回结果
//因为Cache行为上的不确定性和复杂性，如命中与否无法预测，什么时候返回/相应数据无法得知等等，写起代码会发现经常牵扯“意外”遇到“cache”取出
//经验上说，在发出cache请求的下一级经常会有个cache取出数据的缓存，这样做是写起来最舒服的
always @(posedge clk) begin
   if (reset || (ms_ready_go && ws_allowin) || flush_sign) begin
       data_rd_buff <= 32'b0;
       data_buff_enable <= 1'b0;
   end
   else if (data_data_ok && !ws_allowin) begin
       data_rd_buff <= data_rdata;
       data_buff_enable <= 1'b1;
   end
end

//TLBEHI写操作检测，用于与执行级TLBSRCH指令的冲突检测
assign ms_wr_tlbehi = ms_csr_we && (ms_csr_idx == 14'h11) && ms_valid; //stall es tlbsrch

//CACOP指令操作码解析
assign cacop_op = ms_dest;                                              //CACOP操作码存储在目标寄存器字段
assign cacop_op_mode    = cacop_op[4:3];                               //操作模式为操作码高2位
assign cacop_op_mode_di = ms_cacop && ((cacop_op_mode == 2'b0) || (cacop_op_mode == 2'b1));  //DI类型CACOP操作

//用于difftest的地址信息缓存
reg  [ 7:0] tmp_data_index  ;
reg  [ 3:0] tmp_data_offset ;
always @(posedge clk) begin
    tmp_data_index  <= data_index_diff;
    tmp_data_offset <= data_offset_diff;
end

//生成用于difftest的物理地址
assign ms_ld_paddr = {data_tag_diff, tmp_data_index, tmp_data_offset};

endmodule

## 访存阶段关键设计要点

### 1. 数据缓存机制
访存阶段实现了一个重要的数据缓存机制，当cache返回数据但写回级无法接收时，通过`data_rd_buff`和`data_buff_enable`来暂存数据，避免丢失cache的返回结果。这是解决流水线阻塞时数据保持的重要手段。

### 2. 地址转换与页表管理
访存阶段是页表转换的核心实现部分，支持：
- **直接地址模式(DA)**：简单的虚实地址一对一映射
- **分页模式(PG)**：通过TLB进行地址转换
- **DMW窗口**：高速的直接映射窗口，减少TLB查找开销

页表相关的例外检测也在此阶段完成，包括TLB重填、页面无效、特权级检查等。

### 3. Cache控制与非缓存访问
通过`data_uncache_en`信号控制是否绕过cache直接访问内存，支持多种判断条件：
- 直接地址模式下的MAT属性
- DMW窗口的MAT属性  
- TLB转换结果的MAT属性
- 强制禁用cache标志

### 4. 原子操作支持
实现了LL/SC指令的原子操作语义：
- LL指令记录访问地址到`lladdr`
- SC指令检查地址是否匹配，不匹配则取消操作
- 支持与cache miss等情况的交互

### 5. 前递通路优化
访存阶段的前递通路需要特别考虑load指令的特殊性：
- load指令需要等待cache返回才能提供正确的前递数据
- 通过`dep_need_stall`信号指示需要停顿的情况
- 与标准的执行级前递形成完整的冒险解决方案

访存阶段在整个流水线中起到承上启下的关键作用，不仅要处理复杂的内存访问逻辑，还要处理页表转换、例外检测等系统级功能，是理解现代处理器内存子系统的重要环节。
