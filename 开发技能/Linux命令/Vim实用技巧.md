<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/76c7d45adc7620041b2be66817ee97c.jpg" width = 500 /> </div>


<br/>


<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/vi-vim-cheat-sheet-sch1.gif" width = 700 /> </div>


<br/>

<div align="center"> <img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/20160907135321944.png" width = 750 /> </div>


<br/>



# 第 1 章  Vim 解决问题的方式

vim 对重复性操作进行了优化，学会规划按键动作，可以高效利用 vim。

## 技巧 1  结识 `.` 命令

`.` 在 normal 模式下执行，会重复上次的修改。上次的修改可以是很多东西：
- 在 normal 模式：即上次执行的命令
- 在 insert 模式：从进入插入模式那一刻开始，直到返回普通模式，会记录所有修改内容，<span style="background:#fff88f">但是在 insert 模式中移动光标会重置修改状态</span>。

`.` 命令是一个微型的宏，vim 可以录制任意数目的按键操作。



> `x` 命令：删除当前光标下的字符。
> `dd` 命令：删除整行。
> `>G` 命令：增加当前行到文档末尾的缩进层级。
> 
> **光标移动命令**
> - `h`：向左
> - `j`：向下
> - `k`：向上
> - `l`：向上


---

<br/>


<br/>


<br/>

## 技巧 2  不要自我重复

