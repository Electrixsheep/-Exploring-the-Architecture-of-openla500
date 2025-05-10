## 前言

这里是openLa500（一个单发射五级流水线核）代码解读的仓库

openLA500是一款实现了龙芯架构32位精简版指令集（loongarch32r）的处理器核。其结构为单发射五级流水，分为取指、译码、执行、访存、写回五个流水级。并且含有两路组相连结构的指令和数据cache；32项tlb；以及简易的分支预测器。此外，处理器核对外为AXI接口，容易集成。

openLA500源代码见[openLA500](https://gitee.com/loongson-edu/open-la500)

​**​目标读者​**​：CPU 设计初学者、对开源 CPU 实现感兴趣的开发者。


## 导读建议

1. ​**​结合书籍​**​：推荐与《CPU设计实战》同步食用。
    - 先阅读书籍基础章节
    - 再结合本仓库阅读代码解析
3. ​**​动手实践​**​：见[LoongsonEdu](https://gitee.com/loongson-edu)内相关内容，实验环境见[chiplab](https://gitee.com/loongson-edu/chiplab)或[《CPU设计实战 LoongArch版》本地FPGA平台实验环境](https://gitee.com/loongson-edu/cdp_ede_local)，实验手册见[CPU设计实战：LoongArch版](https://bookdown.org/loongson/_book3/)


## 解读内容框架

_(持续更新中...)_
