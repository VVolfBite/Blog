---
title: 知识Linux
date: 2023-10-30 00:00:00
updated: 2023-10-30 00:00:00
description: Linux是很常用的生产环境，归功于其开源特性，Linux比Windows更适合开发生成，不少服务器和应用是基于Linux开发的
categories:
- 知识
- 系统
- Linux
tags:
- Linux
aside: true
top_img: img/a5.png
cover: img/a5.png
---

# 知识Linux

## Unix常用命令

命令的**英文字母大小写敏感**

### 用户登录

root不受限制，可以随意修改和删除文件。root用户可以通过useradd创建新用户，新用户信息包括用户名、ID、主目录。

### 基本命令

```shell
# 联机手册 取自manual
man name(手册名) -k regexp(按正则表达式给出手册)  | q->quit Space->换页 上下箭头->上下换行
# 查询日期 
date "+%Y年 %m月 %d日 %H时 %M分 %S秒 %j该年中第几天 %s从1970年开始的秒数"
# 校对时间
ntpdate 
# 打印日历
cal month year | 不指定则默认当前月
# 计算器
bc -l 指定20位精度 | 输入后就可以使用了 支持简单编程
# 修改口令
passwd | root可以修改任意用户口令，但不能查看其他用户口令，因为系统不存储明文口令而是哈希映射

```

### 了解系统状态

```shell
# 查看系统用户
who | 打印的第一列是用户名   第二列是终端设备文件名
who am i | 打印当前登录用户
whoami | 只打印用户名
# 开机时间
uptime | 分别列出当前时间 开机时间 登录用户数 平均负载(1min 5min 15min)
"12:55:45 up 41 min,  1 user,  load average: 0,14, 0,13, 0,17"
# 列出资源信息
top | 列出进程信息 其表 virt进程逻辑地址空间大小(KB) res驻留内存数 shr共享内存数 %cpu占用cpu比例(可以超过100%) %mem占用内存比 time+ 占用cpu时间
# 查看进程状态 Process status
ps -e 打印所有进程 -f-l 控制列出属性数量 | UID用户ID PID进程号 PPID父进程号 C最近CPU占用 STIME启动时间 SZ逻辑空间大小 WCHAN睡眠通道 TTY终端名 TIME累计占用CPU时间 CMD启动命令 PRI优先级 S睡眠状态S睡眠/R运行/Z僵尸 
# 了解内存情况
free 默认按KB计算 -b 按块计算 -g 按GB计算 -m 按MB计算
# 了解系统负载
vmstat num 指定每隔几秒打印状态 | cpu us=user sy=system id=idle wa=wait
# 查看运行时间
time cmd
```

### 文本文件处理

```shell
重定向 将标准输入/输出重新定向到相关文件 | 管道 将一个命令的输出送入下一个命令输入
命令特点1. 当不指定对象则从标准输入处理 2.指定对象则对对应文件处理 3.大多可以指定多个对象 4.结果默认标准输出
# 逐屏显示内容
more/less | 回车 滚动一行 q 退出 Space 换页 less命令和more类似，但支持更简洁展示
# 列出文件内容
cat -n 打印行号
od -t o1(八进制) 逐个字节打印
echo 向屏幕打印文字
# 打印文件头部/尾部
head/tail -n num(指定显示行数) -f(forever实时打印追加内容)
# 三通
tee 同时输出到stdout和指定文件
# 计数
wc -l(只计算行数) 
# 排序
sort -n(按数字排序)
# 翻译字符
tr string1 string2 把标准输入拷贝到标准输出，将string1对于的字符换成string2
# 筛选重复行(上下紧邻的行)
uniq 只能指定一个文件 -u 只保留没有重复的行 -d 只保留重复的行 -c 计数 默认时删去重复行
```

### 文本文件比较

```shell
# 两文件逐字节比较
cmp file1 file2 | 逐个字节比较内容是否一致，若一致则不给出任何提示，否则打印第一个不同之处
# 文件内容区别判别 
md5sum/sha1sum file1 ... | 给出文件哈希值 md5为16字节hash sha1为20字节hash 比较哈希值确定文件是否一致 
# 文件差异区别
diff -u(unified) file1 file2 | 列出file1如何转化为file2 文件 指令有a(add)、c(change)、d(delete)
n格式为num1,num2cnum3,num4表示将file1中num1-num2的行替换为file2中num3-num4的行 行头的符号：<移出 >移入
u格式为@@ -num1,num2+num3,num4 @@表示将file1中的num1开始num2行删除，增加file2中的num3开始的num4行 行头的符号：-移出 +移入
# 文件版本管理
SCCS CVS SVN GIT
```

### VI编辑器使用


```shell
#用户偏好 $HOME/.exrc 
set number 显示行号
set tabstop = 4 制表对齐

命令整理：
a在光标右边追加 i在光标右边插入 num hjkl控制光标上下左右执行num次 ctrl-b/f 向后/向前翻页
^/$ 移动光标到行首/行尾  num w/b向右/左移动num个单词 :num 移动到num行 %移动到匹配的括号

x删除一个字符 num dd删除num行 rx替换一个字符为x u取消一个操作 .重复一个动作 ZZ存盘 
:wq->存盘退出 :w->存盘 :q!->不存盘退出 
以下均可以使用num1,num2格式指定起始和目标行
:r file->插入文件内容 :w file->写入行至文件file :d->删除行 :y 拷贝到剪贴版  :co复制 :m移动 
p粘贴 j合并当前和下一行 
ctrl-l 刷新 ctrl-g状态显示

/pattern 按模式查找 按n向下查找 按N向上查找
:n1,n2s/pattern/string/g 替换n1-n2行内匹配的模式为字符串 可以使用:^定界从而避免需要对/转义

问题整理：
ctrl-s会进入流量控制从而造成死机现象 按ctrl-q恢复
ctrl-z导致当前进程被挂起 使用jobs查看被挂起的进程使用 %x恢复x号作业
backspace无法使用 原因是行律设置不正确 用stty erase ^H告诉终端将ctrl-h作为删除符
linux使用文本格式与windows不一样 需要格式转换至windows的格式 可以用todos/frodos转换
linux下换行为一个字节 windows为两个字节
在linux下中文为UTF-8（3字节）  在windows下中文为GBK编码（双字节） iconv -f -t 可以实现编码转换

```

### 正则表达式

**正则表达式和文件通配符规则不一样**

六个元字符： . * [ \ ^ $

使用转义字符后可取消元字符含义

* . : 匹配任意单字符

* []: 匹配方括号中任意一个单字符, [] []表示的是匹配一个中括号

* ^ : 表示补集，需要放在[]内开头，否则失去意义

* \* ：使一个**单字符**任意匹配多次

* $/^：锚点 \$在尾部有意义表示必须为结尾 ^在首部有意义 表示必须为开头



* (): 匹配圆括号的集合

* \+:使一个单字符匹配至少一次

* \d 数字 \D非数字

```shell
# 行匹配
grep -F 固定字符串 -G基本正则 -E扩展正则 -P perl正则 -n 显示行号 -v 翻转结果，只打印未被筛选的行 -i 忽略字母大小 reg obj reg可以带单引号以取消文件通配符的替换
egrep 扩展正则表达式
# 流编辑
sed -e '命令1' -e '命令2'  -f 命令文件
s/reg/string/g 可以使用:^定界从而避免需要对/转义 完成匹配ref的字符串替换为string 在reg内使用\(\)可以实现分段编号 \0整个字符串 \1表示第一个 ...
# 逐行扫描
awk '程序' -f '程序名'
程序格式： 条件{动作} 每行表示一个记录内置变量NR行号 每行内空格隔开的叫做域$1,$2域内容$0整个行
		条件可以用c语言运算符或/reg/表示匹配的行
		动作可以指定运算 正则匹配，流程控制，打印print/printf 例如$2 ~ "[1-9]" 表示第二个域要是非0数字
```

### 文件

文件命名规范：只有字节0和斜线/ 不能出现在文件名中；大小写是区分的

* /etc 配置系统信息 passwd用户名称和主目录 hosts域名对应关系 networdk网络配置 rc*.d初始化脚本 profile类似环境变量表
* /tmp 临时目录 用户只可以删除自己的文件
* /var 运行时改变的文件
* /bin 可执行文件
* /usr/bin 其他可运行文件
* /dev 设备文件
* /usr/include 头文件
* /lib /usr/lib 链接库文件 .a静态链接 .so动态链接

文件通配符规则：

* \* 匹配任意长度文件名字符，但.在作为文件名或路径分量第一个字符时必须显示匹配，即使*也无法与之匹配，路径符/也如此
* ？ 匹配任意单字符
* []匹配给定集合任意一字符
* ~匹配当前用户主目录
* .文件 当前目录 ..文件 上一级目录 这不是由该规则处理的
* 处理过程
  * 展开如 cat *.c实际展开为是cat arc.c try.c zap.c
  * 展开时可能导致被展开的文件一个作为输入一个作为输出执行命令而出错
  * 使用单引号可以避免文件通配符处理
  * 展开工作被启动程序不可感知
  * 展开甚至可能被展开成一个参数

文件权限：

* 分为三个等级：文件主、同组用户、其他用户
* 普通文件权限
  * 读R
  * 写W：不可写也可能被删除
  * 执行X：二进制文件/脚本文件(要求第一行指定解释程序,例如#！/bin/sh)
  * SUID权限：权限位为S，当其他用户执行其程序时，其获得文件主的权限；实际UID为当前用户而有效UID为文件主，这样就可以通过编写一个程序访问相关文件
* 目录文件权限
  * 读R：是否允许访问目录
  * 写W：文件夹能否增加/删除/移动/修改文件名字等 但文件夹下的文件仍然可以修改
  * 执行X：分析路径名可否检索目录
  * 黏着位T：文件是否可被其文件主删除
* 若文件主人和进程主人相同则使用文件主权限；若文件主人和进程主人不相同，则比较文件组号和进程组号是否相同，相同则使用组用户；都不是则用其他用户
* 超级用户不受制约





```shell
#文件名列表
ls -F 在目录后加/ 在可执行文件后加* 在符号连接文件后加@  -l 长格式 "文件类型(一位) 访问权限(2-4所有者 5-7同组用户 8-10其他用户) 文件link数 文件所属用户名 文件所属用户组名 文件大小/目录表大小(字节) 最后修改时间 文件名" -h 切换大小 -d 遇到目录只打印目录信息 -a 打印所有文件，这样可以把.隐藏文件打印 -s 打印磁盘空间 -i 打印i节点号| 可以0-n个实参 不带则打印当前目录 是文件则列出文件 是目录则列出目录下文件
# 文件复制
cp file1 file2  把file1拷贝为file2
cp file1 ... dir 把一系列文件拷贝到dir目录
cp -r 递归复制 -v 冗长方式 -u 增量拷贝，名字相同比较时间戳
# 文件移动
mv file1 file2 把file1移动到file2 ，相当于改名或覆盖
mv file1 ... dir 把一系列文件移动到dir目录
mv dir1 dir2 改名dir1为dir2
# 文件删除
rm file1 ... 删除一系列文件 -r 递归删除文件 -i 每次删除都要确认 -f 强制删除无提示
# 打印当前工作目录
pwd
# 改变工作目录
cd
# 创建/删除目录
mkdir -p 创建原本不存在的目录
rmdir 要求目录下无一般文件才能执行
# 修改最后一次时间
touch 
# 遍历目录树
find dir1 ... -name '匹配的文件名(注意有引号防止通配符替换)' -regex pattern 给定正则表达式匹配 -type 给定文件类型  -size +/-nc/b/k/M 要求文件大小 -mtime +/-ndays 指定文件修改时间 - newer file 要求比文件file新 
注意要在分号左边有斜线/;
-print 打印动作 -exec 执行至分号的命令，用{}表示查到的结果  -ok表示确认 用\(con1 -o con2 \) 表示或
# 标准输入追加参数表
xargs -n 指定批处理量 追加标准输入至命令尾作为参数 常用于管道机制
# 文件压缩/解压
tar cvf file file1...filen 将当前目录备份/压缩到file
tar tvf file 查看文件含有的文件
tar xvf file 解压文件
-z 按gz压缩 -j 用bz2压缩
gzip/gunzip 也可以压缩/解压
bizp2/bunzip2 也可以压缩/解压
使用--显式告知命令选项结束了
# 修改权限
chomod [ugoa][+-=][rwxst] file1..file2
u=user g=group o=other a=all +赋予 -撤销 =设置 r=read w=write x=exe s=filestick t=dirstick
还可以使用数字每一位表示
只允许文件主人和超级用户修改权限
修改文件./..权限需要对其有读写权限
# 决定进程创建文件初始权限
umask xxx |不给xxx则是打印权限 设置的是掩码 意为取消位为1的权限
注意:其权限设置是同时受到创建其的函数和umask过滤的效果的
```

### 命令获取参数和参数风格

程序获取参数的方式：

* 配置信息：系统和用户级别偏好设置

* 环境变量
  * 使用env打印相关环境变量 
  * 使用export NAME=XXX输出环境变量 
  * 程序内可以使用getenv获取环境变量

* 命令行参数   int argc 指数组里有效的参数  char **argv存储命令参数
  * dd格式 采用param=value格式
  
  * 短命令格式 -options value
  
  * 长短选项格式 -soptions value --loptions values 其中短选项一般可以放在一起或放在后面 使用--显式终止选项
  
    

## Shell脚本设计

### Shell机制

Linux使用的是B-Shell，但也可以用C/K-Shell，其主要工作是命令解释器，还有一些替换工作和编程能力。

主要用途：批处理，比算法语言效率低；其是面向命令处理的语言；采用策略和机制相分离



启动Bash的三种办法：

* 注册式：启动时自动执行.bash_profile 退出时自动执行.bash_logout
* 交互式：启动时自动执行.bashrc
* 脚本启动

脚本执行：

* bash<sh无法携带命令行参数 新Shell
* bash -x sh打印运行轨迹执行脚本 新Shell
* 为脚本增加可执行文件 chmod u+x sh 然后执行 新Shell
* . sh .说明在当前shell运行 原Shell
* source sh 将sh在当前shell运行 原Shell

历史

* 上下箭头
* !!引用上一条命令
* !str以str开头的最近用过的命令

别名

* 使用alias new_command='old_command' 可以将常用命令简化为新的简洁名

Tab补全

* 首个单词补全$PATH
* 后续单词匹配ls ./

输入重定向stdin fd=0

* 使用<file 将标准输入重定向到file
* 使用<<word 将标准输入重定向到脚本文件直到再次出现定界符(不包括定界符)，使用单引号取消效果
* 使用<<<word 将标准输入重定向到后续输入

输出重定向stdout fd=1

* 使用>file 将标准输出覆盖到文件
* 使用>>file 将标准输出追加到文件
* 使用2>file 将错误输出重定向
* 2>&1将标准错误重定向到标准输出

管道

* 只是将标准输出重定向到了下一个命令

### Shell编程

#### 变量

只有字符串类型，命名规则与C一致

赋值是**一条单独命令**，因此等于号不要有空格，赋值内容也不要有空格，有则需要用双引号括起来

引用使用$var即可，其效果是替换，**没有则为空**，使用set -u时可以使引用未定义报错，使用set +u恢复

变量的局部性是相对Shell而言的。

read name实现输入赋值

使用eval实现输入命令，例如read line; eval "$line" 会将输入的语句作为命令执行

**常见内部变量**：

* $0表示脚本文件本身名字
* $1 ...表示命令行参数
* $# 表示行参数个数
* "$*" 等同于"\$1 \$2 ..."
* "$@" 等同于"\$1 " "\$2" "\$3"...
* shift命令可以移位一个参数
* $?上一次命令返回码

**常见环境变量**:

* HOME 当前用户主目录
* PATH 命令查找路径
  * 它不搜索当前目录，只按顺序搜索其值的目录 若填入.至变量是危险的
* TERM 终端类型

使用env查询环境变量，使用set查询局部变量

#### 替换

Shell先替换命令再执行命令

* 文件名生成(展开通配符)
* 变量替换(将$var替换为该变量的值)，替换时双引号与单引号消失
* 命令替换( 使用反撇号用命令的标准输出替换，也可以用$(com)来操作 )  可以利用expr来达到运算

#### 元字符

使用 \ 取消元字符本义，若其后不是元字符则没有意义

**注意转义嵌套，如shell到正则双转义**

* 空格/制表符 用于参数分割
* 回车 用于执行命令
* <>| 重定向与管道
* ; 用于一行输入多个命令
* & 后台运行
* $引用变量
* ` 命令替换，中间出现的反撇号和反斜线应该被转义
* *[]? 文件通配符
* ( )定义函数
* ' '单引号 取消所有元字符的含义
* " "双引号 取消除了$和`和' 和 ” '的其他所有元字符含义，中间出现的美元号、反撇号、引号应该被转义

#### 逻辑判断

**仅依靠命令返回码**，即main函数返回值决定命令是否执行成功。若main没有返回任何值则返回码为任意值，若进程以exit(code)退出，则code为返回码。

**内部变量 $?表示上一次命令的返回码**。

利用**短路特性**实现条件选择执行：

* cmd1 && cmd2 : 若cmd1成功执行则执行cmd2，否则不执行

* cmd1 || cmd2 : 若cmd1不成功执行则执行cmd2，否则不执行

为了更好配合 使用**/bin/true 与 /bin/false**，前者永远返回0，后者永远返回非0

```shell
# 条件判断
test -r/w/x 测试文件是否可读/写/执行 -s 检测文件大小 -f/d检测文件/文件夹 str1 = str2 判断字符串相等，注意空格与str两侧引号  -eq/ne/gt/ge/lt/le大小等于 !/-o/-a 非或与
例如 
	9=0 为真（字符串不空）， 9 = 0 为假
	a="" [ $a = "" ] 结果是错误的，因为$a被替换为空
[   | 与test完全一致，不过要求最后一个参数为] 
```

#### 命令组合

* 使用{ list; } 打包命令，效果是命令在原来Shell中执行，**{ }被视为一个命令 因此左花括号后必须存在一个空格，右花括号前空一格应该有分号;以分割命令**；若多行书写可以不用
* 使用(list) 打包命令，效果是命令在新的子Shell中执行

#### 表达式运算

```shell
# expr命令 
# 注意空格分割参数 另外若其为元字符需要转义防止Shell做特殊处理
( )
+ - * / %
> >= < <= = !=
| &
: | 使用方法expr string : pattern 时刻记住应该对string加双引号防止string为空导致报错
```

#### 条件分支

```shell
if condition
	then list
elif condition
	then list
else
	list
fi
注意：if/then/elif/then/fi被视为一个命令，因此需要注意空格且保证其前没有其他字符 
	所以 if 与 then 不要在同一行
	
case word in
	pattern1) 
		list1
		;;
	pattern2) 
		list2
		;;
	...
esac
注意: ;;是一个整体类似break，不要在中间加空格 word与pattern匹配则执行相关命令，其按照文件通配符规则匹配 可以使用竖线表示多个模式

```

#### 循环

```shell
while condition
	do 
		list
	done
注意：do与done匹配 
for name in word1 word2 ...
	do list
	done
for name #相当于 in $1 $2 ... 
	do list
	done
为了实现类似C的从0到n的循环可以写：
for i in `seq 1 $n`
	do list
	done
引用时记住用$i

break continue exit
break可以使用break 2 打破两层循环
exit n 指定脚本程序结束的返回码
```

#### 函数

```shell
name() {list;}
注意：函数的效果实际也是命令 函数定义完毕相当于一条名为name的自定义命令定义
参数传递：传参直接用一般命令传参即可
参数使用：在定义时使用$1 $2等表示第几个参数
返回值：使用return 返回
内部变量：内部变量相当于局部变量 作用域不限制在函数内
```



## 进程控制与通信

### 进程概述

程序是指指令与数据的集合，进程指的是"实例化"到内存的执行的程序。

程序用于初始化进程的指令段和数据段，初始化结束逻辑上就与进程再无联系，但实际操作系统可能不会一次初始化完毕，所以进程运行时程序文件还是不可以删除/修改的。

**进程组成部分**

* 指令段：CPU指令代码——大小固定不变且只读
* 用户数据段：全局变量、静态变量、字符串常数等 ——允许数据段增长与缩小.内存动态分配
* 用户栈段：函数( 包括主函数，主函数参数在栈底 )如函数内部变量、函数参数，返回地址——允许动态增长，存在增长限制
* 系统数据段：内核数据
* 命令行参数和环境参数：位于堆栈底部的初始化数据，可以通过environ/getenv或者main第三个参数获取


**进程执行状态**

* 运行状态：使用CPU资源，时间记账分别记为用户时间和系统时间，总时间为等待时间和前两者相加，其指从任务启动到结束的时间
* 就绪状态：等待CPU资源
* 睡眠(阻塞)状态：不需要CPU资源，睡眠状态被叫醒一般赋予高优先级，而高优先级时间片一般较短、低优先级时间片较长

应当避免忙等待，即使循环等待任务，每次睡眠10毫秒都能有效减少浪费。

#### 进程的系统数据

* 页表
* 打开的文件描述符
* 核心态堆栈————————————————————————————————————————————
* 进程状态
* PID\PPID\UID\GID\进程组组号\umask

现在一般而言执行单位为线程，资源单位为进程

**虚拟到物理内存转换**


#### 进程生命周期

* 创建新进程：fork
  * fork系统调用是创建新进程的唯一方式
  * 完全复制：指令段、用户数据段、堆栈段     部分复制：系统数据段
  * **返回值： 父进程返回值一定大于0，其返回的是子进程的PID   子进程返回值一定等于0   失败时返回-1**
  * 从fork的位置父子进程开始继续执行
  * 内核上创建新的PCB，复制父进程环境（PCB与内存资源）给子进程
  * 使用COW可以实现共享初始内存，使得只有在写时进行复制
* 用指定程序重新初始化一个进程：exec
  * exec调用是破坏当前进程数据，使用指定的可运行文件重新填写并执行，只要执行成功了则原程序后续代码没有任何意义
  * 可选格式：l-list v-vector 指定命令行参数/e-env p-path
  * execl / execv / execle /execve /execlp /execvp
  * 使用int system(char *string)可以实现使用shell的管道机制、替换机制，其效果是启动shell执行命令
* 结束—僵尸进程
  * 子进程运行结束，其死亡以后应该报告给父进程，这时会留下子进程的一部分于其PCB，等待父进程收尸
  * 当父进程收尸完毕，则僵尸进程消失
  * 僵尸进程占用资源很少，只占用PCB资源，若不收尸会导致进程表用光
  * 如果父进程先于子进程结束，则子进程称为孤儿进程，此时子进程成为系统的子进程，死亡就不再留下尸体
  * **使用wait等待子进程终止，pid_t wait(int *stat)其中stat是传入传出参数，用于记录子进程死亡情况，TERMSIG说明被杀死，EXITSTATUS说正常死亡**

### 文件描述符

* 磁盘文件：文件名与i节点表
* 活动文件目录——打开的文件
  * 文件描述符表FDT：每个进程一张，记录进程打开的文件或设备
  * 系统文件表SFT：整个系统一张，每条记录着活动文件的读写操作、引用计数、读写位置指针、内核inode下标
  * 活动i节点表inode：整个系统一张
  * 文件描述符表——系统文件表——活动inode节点表
  * 使用fork会让父子进程的FDT指向同一个SFT，也就会使用同一个位置指针
  * 在open使用O_CLOEXEC或者fcntl 设置close_on_exec 使得执行exec时自动关闭文件

#### 一些机制

* **重定向——使用int dup2(int fd1,int fd2)将文件描述符fd1替换到fd2，相当于使用fd2时在用fd1**
* **管道**
  * 创建管道——int pipe(int *pfd)，主要是为了两进程间通信
    * 创建一个管道，传入两个文件描述符，其中pfd[0]为读端、pfd[1]为写端，效果是不用使用open的打开两个文件
  * 管道写——ret= write(pfd[1],buf,n)
    * 管道满则阻塞，返回值是实际写入量
  * 管道读——ret=read(pfd[0],buf,n)
    * 管道空则阻塞，返回值是实际读出量
  * 管道关闭——close
    * 关闭读端，写端将收到SIGPIPE信号并导致进程结束，即使捕获信号不死亡也会返回-1
    * 关闭写端，读端会返回0
  * 注意
    * 管道传输的是没有边界的字节流
    * 父子进程要双向通信则最好用两个管道，防止自己读自己写的内容
    * 管道可能导致死锁
  * 命名管道：命名管道实际上类似一个文件
    * 创建：命令mkmod pipename p
    * 发送：open打开写即可
    * 接收：open打开读即可
* 信号
  * kill kill命令用于向进程发送信号 即int kill(int pid ,int sig)
    * 向pid的进程发送信号sig，其中若pid=0则指本进程，若pid<0则向-pid所在组发送信号
    * sig=0可以用于测试pid是否存在，通过捕获ESRCH即可
  * pause()/int alarm(int secs) 当信号到达前进程睡眠；alarm会导致子进程继承相关报警值，secs=0则关闭报警信号
  * 可以使用全局跳转处理信号防止堆栈累计
    * signsetjmp(env,1) 与signlongjmp(env,1)使执行longjmp直接跳转回setjmp
  * SIGSEGV 段违规信号：访问了进程不该访问的信息时
  * SIGFPE 零做除数：进程尝试使用0做除数
  * SIGPIPE 管道关闭： 读端关闭管道
  * SIGKILL 无条件自杀
  * SIGTERM 软件终止
  * SIGHUP 挂断，如关闭终端
  * SIGINT 中断，按下ctrl-c时发送
  * SIGCLD 通知子进程终止 ，将其signal(SIGCLD,SIG_IGN)忽略掉子进程死亡可以
  * 进程一般在收到信号时死掉，有的会产生core文件；也有程序会忽略信号；程序员可以编写捕捉信号并继续执行
    * 按照默认处理：signal(SIGINT,SIG_DFL)
    * 按照忽略处理：signal(SIGNINT,SIG_IGN) 忽略会作为属性被继承
    * 按捕捉处理: signal(SIGNINT,func) 将信号按照func处理
  * 执行scanf\sleep\msgrcv\read\write等会导致进程睡眠，当进程收到信号就会被惊醒，相关系统调用失败并返回-1；若进入深度睡眠，则不会被信号打断，即使SIGKILL也无法杀死进程

### 进程间协作

* 信号灯
  * 创建——int semget(int key,int nsems ,int flag)
    * 创建或获取已有的信号灯，返回信号灯组
    * nsems指定个数
  * 删除——int semctl(int sem_id,int snum,int cmd,char* arg)
    * IPC_RMID删除信号灯
  * 操作——int semop(int sem_id,struct sembuf *ops,int nops)
    * ops是操作数组 分别指定编号 操作，操作选项 sem_op<0 P操作 >0V操作 =0等待直到为负数
* 共享内存

### 内存映射文件

将文件一部分连续内容映射成内存，这样修改文件不用再以扇区修改或访问，将来修改文件时统一写回即可。

* void *mmap(void *addr ,size_t len ,int prot ,int flag,int fd ,off_t offset)
  * 选择一个地址将文件fd在offset的len个长度映射到内存，返回其映射地址
  * 可以实现共享内存的效果
* 

## Socket编程

### Socket概述

Socket接口面向网络通信不仅仅用于TCP/IP,其可以使用虚拟loopback(127.0.0.1)实现同一台计算机自己通信。

* TCP：面向连接、可靠(收到的数据一定保证正确)、字节流传输
* UDP: 面向数据报，不可靠(丢报、乱序、被流量控制)、广播组播

网络字节采用先发送高位再发送低位的字节顺序，因此可能需要htonl ntohl进行字节转换

#### Socket编程

一些理解：

* socket函数——socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)

  * socket本身设计考虑了多种网络，因此使用时必须指明 网络族 通信方式 传输协议
  * 效果是创建并返回了一个指定类型的"文件"，后续通信的读写都相当于在对文件操作包括关闭连接等
  * 这个时候端点还未指定
  * 英文原文：int socket(int _domain, int _type, int _protocol) noexcept(true)
    Create a new socket of type TYPE in domain DOMAIN, using
    protocol PROTOCOL. If PROTOCOL is zero, one is chosen automatically.
    Returns a file descriptor for the new socket, or -1 for errors.

* bind函数——bind(sock, (struct sockaddr *)&name, sizeof(name))

  * bind 指定本地端点名，也可以在客户端程序
  * 效果是告知参数中 指定的端点name 将信息发往sock这个文件中来
  * 若不指定则相关函数需要本地端点时会自己选择
  * 英文原文：int bind(int \_fd, const sockaddr \*_\_addr, socklen_t \_len) noexcept(true)
    Give the socket FD the local address ADDR (which is LEN bytes long).

* listen函数——listen(sock, 5);

  * listen 开始监听到达端点的请求，并给内核一个通知

  * 效果是 在 打开的"文件"sock中发现送来的连接建立请求

  * 英文原文：int listen(int \__fd, int \__n) noexcept(true)  

    Prepare to accept connections on socket FD.
    N connection requests will be queued before further requests are refused.
    Returns 0 on success, -1 for errors.

* accept函数——data_sock = accept(sock, 0, 0);

  * accept  在指定端点处接受连接请求，其需要完成握手的部分工作

  * 在第三次握手结束时accept会返回

  * accept会无限等待这会和read效果冲突

  * 效果是 接受在打开的"文件"sock中发现的连接建立请求并返回一个"文件"data_sock，后续使用这个文件进行通信

  * 英文原文：int accept(int \_fd, sockaddr *_restrict_ _addr, socklen_t *_restrict _addr_len)
    Await a connection on socket FD.
    When a connection arrives, open a new socket to communicate with it,
    set *ADDR (which is *ADDR_LEN bytes long) to the address of the connecting
    peer and *ADDR_LEN to the address's actual length, and return the
    new socket's descriptor, or -1 for errors.

    This function is a cancellation point and therefore not marked with
    __THROW.

* connect函数——connect(sock, (struct sockaddr *)&name, sizeof(name))

  * connect  向指定端点处发起一个连接请求，其需要完成握手的部分工作

  * 在第二次握手结束时connect会返回

  * 效果是 向指定地点name发起连接，连接的通信管道或者说为了交互而使用的文件为sock

  * 英文原文：int connect(int _fd, const sockaddr *\_addr, socklen_t _len)
    Open a connection on socket FD to peer at ADDR (which LEN bytes long).
    For connectionless socket types, just set the default address to send to
    and the only address from which to accept transmissions.
    Return 0 on success, -1 for errors.

    This function is a cancellation point and therefore not marked with
    __THROW.
  
* select函数——select(int maxfdp1, fd_set *rfds ,fd_set *wfds, fd_set *efds ,struct timeval *timeout)

  * select 等待三个集合中任意一个准备完毕时并返回
  * 参数分别的含义是 等待的最大文件描述符号+1 ， 读文件描述符集合 ， 写文件描述符集合 ，错误文件描述符集合 ， 超时时间 ；这些参数可以是NULL ，若是集合为NULL表示我们不关注这种信息，若时间为NULL代表永不超时
  * 参数是传入传出型参数，select返回时就会修改相关集合，仍在集合中的描述符即为准备好了;返回值表示有几个符准备好了
  * 对于准备完毕的含义：一旦缓冲区可用即可，对于错误只有TCP加急数据到达才算异常，对于连接关闭、网络故障等都不算异常情况

* 其他收发函数

  * int recv(int sockfd,void *buf,int nbyte,int flags);
  * int recvfrom(int sockfd,void *buf,int nbyte,int flags, struct sockaddr *from,int *fromlen);
  * int send(int sockfd,void *buf,int nbyte,int flags);
  * int sendto(int sockfd,void *buf,int nbyte,int flags, struct sockaddr *to,int *tolen);
  * windows中只能使用recv和send


### TCP

#### 客户端

```c++
//客户端例子
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <unistd.h>
#include <stdio.h>
#include <string.h>
int main(int argc, char **argv)
{
    int sock, len;
    struct sockaddr_in name; // 用于填入地址信息
    char sbuf[SIZE];

    if (argc < 2)
        ; //...
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    // AF_INET Address_Family为因特网 Sock通信方式为流式，协议选择TCP协议
    // 这里创建了一个Socket TCP文件描述符 ，小于0创建失败
    if (sock < 0)
        ; //...

    name.sin_family = AF_INET;          // 地址类型指定地址为因特网
    inet_aton(argv[1], &name.sin_addr); // 地址IP指定为输入的IP地址
    name.sin_port = htons(PORT_NO);     // 地址端口指定为协商好的PORT_NO
    if (connect(sock, (struct sockaddr *)&name, sizeof(name)) < 0)
    // connect参数分别为 文件描述符 地址结构 地址结构长度 ，函数会自动填写本地端点信息，并向填入的目标地址建立连接
    // 函数会导致程序阻塞直到结果返回
    // 这里相当于将Socket TCP当成文件 进行流输入输出了 , 小于0连接失败
    {
        perror("\nconnecting server stream socket");
        return 1;
    }
    printf("Connected.\n");

    //将创建的Socket TCP当成文件进行读写即可
    for(;;)
    {
        if(fgets(sbuf,sizeof(sbuf),stdin) == NULL)
        //发送数据：发送速率大于通信速率则进程被阻塞
            break;
        if(write(sock,sbuf,strlen(sbuf))<0)
        //write函数写入文件失败会返回负数，这里利用这个特性判断Socket是否被关闭
        {
            perror("sending stream message");
            return 1;
        }
    }
    close(sock);
    printf("Connection closed.\n\n");
    return 0;
}


```

#### 服务器

```c++
//服务端例子: 单进程阻塞
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <unistd.h>
#include <stdio.h>
#include <string.h>
int main(void)
{
    int admin_sock, data_sock;
    struct sockaddr_in name;
    char buf[SIZE];
    int nbyte, i;

    admin_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    // AF_INET Address_Family为因特网 Sock通信方式为流式，协议选择TCP协议
    // 这里创建了一个Socket TCP文件描述符 ，小于0创建失败
    name.sin_family = AF_INET;         // 地址类型指定地址为因特网
    name.sin_addr.s_addr = INADDR_ANY; // 地址IP指定为任意地址
    name.sin_port = htons(PORT_NO);    // 地址端口指定为协商好的PORT_NO
    bind(admin_sock, (struct sockaddr *)&name, sizeof(name));
    // 向admin_sock给出本地端点，这里这么做似乎没有什么意义，它的作用只是告诉服务器尝试在自己拥有的且被指定的所有网络地址尝试监听对应端口：比如说一台机器同时有两个IP那么接下来的监听就会到两个IP上都进行指定端口的监听
    listen(admin_sock, 5);
    // 监听admin_sock，不会造成阻塞

    data_sock = accept(admin_sock, 0, 0);
    // 接受admin_sock收到的连接请求，会导致系统进入阻塞直到建立连接 其返回连入的Socket
    printf("Accept connection\n");

    for (;;)
    {
        nbyte = read(data_sock, buf, SIZE);
        if (nbyte == 0)
        //如果文件，即对方Socket连接断开，则返回值为0；如果无数据则进入阻塞；如果有数据则读出。
        {
            printf("*** Disconnected.\n");
            close(data_sock);
            return 0;
        }
        for (i = 0; i < nbyte; i++)
            printf("%c", buf[i]);
    }
}
```

```c++
//服务端例子: 多进程
//主进程
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <sys/signal.h> //定义信号

#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(void)
{
    int admin_sock, data_sock, pid, name_len;
    struct sockaddr_in name, peer;

    admin_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (admin_sock < 0)
    {
        perror("Create stream socket");
        return 1;
    }
    // AF_INET Address_Family为因特网 Sock通信方式为流式，协议选择TCP协议
    // 这里创建了一个Socket TCP文件描述符 ，小于0创建失败
    name.sin_family = AF_INET;         // 地址类型指定地址为因特网
    name.sin_addr.s_addr = INADDR_ANY; // 地址IP指定为任意地址
    name.sin_port = htons(PORT_NO);    // 地址端口指定为协商好的PORT_NO
    if (bind(admin_sock, (struct sockaddr *)&name, sizeof(name)) < 0)
    {
        perror("binding stream socket");
        return 1;
    }

    listen(admin_sock, 5);   // 监听指定端口发来的连接建立请求
    signal(SIGCLD, SIG_IGN); // 必须执行操作，否则进程僵尸

    for (;;)
    {
        name_len = sizeof(peer);
        data_sock = accept(admin_sock, (struct sockaddr *)&peer, &name_len);
        //接受admin_sock内受到的连接建立请求，并将建立方的地址信息存入peer中，返回一个文件data_sock以实现通信
        if (data_sock < 0)
            continue;
        printf("Accept connection from %s : %d \n",
               inet_ntoa(peer.sin_addr), ntohs(peer.sin_port));
        pid = fork();//创建一个子进程
        if (pid > 0) // 父进程
        {
            close(data_sock);//关闭这个多余的文件描述符，如果不关闭，则以后新来的请求都会导致程序打开一个文件
        }
        else if (pid == 0)//子进程
        {
            char fd_str[16];
            close(admin_sock);//子进程也关闭父进程的文件符
            sprintf(fd_str, "%d", data_sock);//将分配给与对方通信的data_sock准备作为参数
            execlp("./2-1-s", "./2-1-s", fd_str, NULL);//执行程序其处理data_sock这个文件即可
            perror("execlp");
            return 1;
        }
    }
}

// 服务端例子: 多进程
// 子进程
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <sys/signal.h> //定义信号

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv)
{
    int i, nbyte, sock = strtol(argv[1], 0, 0);
    struct sockaddr_in peer, local;
    socklen_t name_len = sizeof(peer);

    char buf[SIZE];

    getpeername(sock, (struct sockaddr *)&peer, &name_len);//获取对方的端点
    getsockname(sock, (struct sockaddr *)&local, &name_len);//获取自身的端点
    for (;;)
    {
        nbyte = read(sock, buf, SIZE);
        printf("%s : %d => ", inet_ntoa(peer.sin_addr), ntohs(peer.sin_port));
        printf("%s : %d", inet_ntoa(local.sin_addr), ntohs(local.sin_port));
        if (nbyte <= 0)
        //连接断开
        {
            printf("*** Disconnected.\n");
            close(sock);
            return 0;
        }
        for (i = 0; i < nbyte; i++)
            printf("%c", buf[i]);
    }
}
```

```c++
// 服务端：单进程
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <sys/signal.h> //定义信号

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define syserr(prompt)  \
    {                   \
        perror(prompt); \
        return 1;       \
    }
static int receive_data(int sock)
{
    char rbuf[SIZE];
    struct sockaddr_in peer;
    int i, nbyte;
    socklen_t name_len = sizeof(peer);

    nbyte = recv(sock, rbuf, SIZE, 0);
    if (nbyte < 0)
    {
        perror("receiving stream packet");
        return 1;
    }
    getpeername(sock, (struct sockaddr *)&peer, &name_len);
    printf("%s : %d", inet_ntoa(peer.sin_addr), ntohs(peer.sin_port));
    if (nbyte == 0)
    {
        printf("*** Disconnected.\n");
        return 0;
    }
}
int main(void)
{
    int admin_sock, data_sock, ret, n, fd;
    struct sockaddr_in name;
    fd_set fds, rfds;

    admin_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (admin_sock < 0)
        syserr("Create socket");

    name.sin_family = AF_INET;
    name.sin_addr.s_addr = INADDR_ANY;
    name.sin_port = htons(PORT_NO);
    if (bind(admin_sock, (struct sockaddr *)&name, sizeof(name)) < 0)
        syserr("Binding socket");

    listen(admin_sock, 5);
    printf("ready\n");

    n = admin_sock + 1;//n用于计数最大描述符+1
    FD_ZERO(&fds);//清空fds
    FD_SET(admin_sock, &fds);//将admin_sock文件描述符加入fds中

    for (;;)
    {
        memcpy(&rfds, &fds, sizeof(fds));//因为select是传入传出型，我们需要将相关的描述符重新抄入
        ret = select(n, &rfds, 0, 0, 0);//只有相关事件准备好了才会继续执行程序
        if (ret < 0)
            syserr("select");
        if (ret == 0)
            continue;

        if (FD_ISSET(admin_sock, &rfds))//如果admin_sock在里边，说明有人向其写入数据了，一般说只可能是连接数据
        {
            data_sock = accept(admin_sock, 0, 0);//处理连接请求
            if (data_sock < 0)
                syserr("accept");
            FD_SET(data_sock, &fds);//将这个连接使用的通信通道加入我们的等待集合中
            if (n <= data_sock)
                n = data_sock + 1;//增加大小
        }
        for (fd = 0; fd < n; fd++)//遍历检查一遍是否有业务通信通道准备读取
            if (fd != admin_sock && FD_ISSET(fd, &rfds))
            {
                ret = receive_data(fd);//读出信息，若连接断开则关闭通道
                if (ret == 0)
                {
                    close(fd);
                    FD_CLR(fd, &fds);//将其从集合删除
                }
            }
    }
}

```

### UDP

#### 客户端

```c++
// 客户端
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <sys/signal.h> //定义信号

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int main(int argc ,char **argv)
{
    int sock,len;
    struct sockaddr_in name;
    char buf[SIZE];
    sock=socket(AF_INET,SOCK_DGRAM,IPPROTO_UDP);//这里使用数据报通信方式和UDP协议
    name.sin_family=AF_INET;
    inet_aton(argv[1],&name.sin_addr);
    name.sin_port=htons(PORT_NO);
    connect(sock,(struct sockaddr *)&name,sizeof(name));//这里的connect没有实际的连接动作，或者说connect只是在内核将name填入sock之中达到了指定默认地址的效果
   	for(;;)
    {
        if(fgets(buf,sizeof(buf),stdin)==NULL)
            break;
        len=write(sock,buf,strlen(buf));
        if(len<0)
        {
            perror("sending message");
            break;
        }
        printf("send %d bytes\n",len);
    }
    close(sock);
}
```

```c++
//服务端
#define SIZE 8192
#define PORT_NO 12345
#include <sys/types.h>  //提供数据定义
#include <sys/socket.h> //提供socket函数及数据结构
#include <netinet/in.h> // 定义数据结构socketaddr_in
#include <arpa/inet.h>  //定义inet_addr函数
#include <sys/signal.h> //定义信号

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int main(void)
{
    
    int sock,len;
    struct sockaddr_in name;
    char buf[SIZE];
    sock=socket(AF_INET,SOCK_DGRAM,IPPROTO_UDP);//这里使用数据报通信方式和UDP协议
    name.sin_family=AF_INET;
   	name.sin_addr.s_addr=INADDR_ANY;
    name.sin_port=htons(PORT_NO);
    bind(sock ,(struct sockaddr*)&name,sizeof(name));//仍然这个绑定只是为了让任何IP的12345端口将数据发送至sock中
    
    struct sockaddr_in peer;
    socklen_t addr_len;
    char buf[SIZE];
    
    for(;;)
    {
        addr_len=sizeof(peer);
        len=recvfrom(sock,buf,sizeof(buf),0,(struct sockaddr*)&peer,&addr_len);
        //因为使用read是无法区分对方地址的，UDP中一般使用recvfrom与sendto
        if(len<0)
        {
            perror("recfrom");
            continue;
        }
        printf("Received %d bytes from %s : %d\n",len,inet_ntoa(peer.sin_addr)
              ,ntohs(peer.sin_port));
        sendto(sock,buf,len,0,(struct sockaddr *)&peer,sizeof(peer));
    }
}
```

