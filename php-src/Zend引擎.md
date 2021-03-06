先看图

![](http://www.php-internals.com/images/book/chapt07/07-01-01-zend-vm.png)





必须要讲的概念

1.解析器是[parser](http://en.wikipedia.org/wiki/Parsing)，而解释器是[interpreter](http://en.wikipedia.org/wiki/Interpreter_(computing))。两者不是同一样东西，不应该混用。 



1.1前者是**编译器/解释器的重要组成部分**，也可以用在**IDE之类的地方**；其主要作用是进行语法分析，提取出句子的结构。广义来说输入一般是程序的源码，输出一般是[语法树（syntax tree，也叫parse tree等）](http://en.wikipedia.org/wiki/Parse_tree)或[抽象语法树（abstract syntax tree，AST）](http://en.wikipedia.org/wiki/Abstract_syntax_tree)。



进一步剥开来，广义的解析器里一般会有**扫描器（scanner，也叫tokenizer或者lexical analyzer，词法分析器），以及狭义的解析器（parser，也叫syntax analyzer，语法分析器）。**扫描器的输入一般是文本，经过词法分析，输出是将文本切割为单词的流。**狭义的解析器输入是单词的流，经过语法分析，输出是语法树或者精简过的AST。**

 ![](http://dl.iteye.com/upload/attachment/157319/5a444555-d71e-3934-b93f-528cd7f737ce.png)



1.2后者则是实现程序执行的一种实现方式，与编译器相对。它直接实现程序源码的语义，**输入是程序源码，输出则是执行源码得到的计算结果；编译器的输入与解释器相同，而输出是用别的语言实现了输入源码的语义的程序。**通常编译器的输入语言比输出语言高级，但不一定；也有输入输出是同种语言的情况，此时编译器很可能主要用于优化代码。 

![](http://dl.iteye.com/upload/attachment/157321/6dd9e63e-0567-355d-8498-a241cc463ef6.png)





2.解释器就是个黑箱，输入是源码，输出就是输入程序的执行结果，对用户来说中间没有独立的“编译”步骤。这非常抽象，内部是怎么实现的都没关系，只要能实现语义就行。你可以写一个C语言的解释器，里面只是先用普通的C编译器把源码编译为in-memory image，然后直接调用那个image去得到运行结果；用户拿过去，发现直接输入源码可以得到源程序对应的运行结果就满足需求了，无需在意解释器这个“黑箱子”里到底是什么。 
实际上很多解释器内部是以**“编译器+虚拟机”**的方式来实现的，<u>**先通过编译器将源码转换为AST或者字节码，然后由虚拟机去完成实际的执行**</u>。**所谓“解释型语言”并不是不用编译，而只是不需要用户显式去使用编译器得到可执行代码而已**。 



3.那么虚拟机（[virtual machine](http://en.wikipedia.org/wiki/Virtual_machine)，VM）又是什么？在许多不同的场合，VM有着不同的意义。如果上下文是Java、Python这类语言，那么一般指的是高级语言虚拟机（high-level language virtual machine，HLL VM），其意义是实现高级语言的语义。VM既然被称为“机器”，**一般认为输入是满足某种指令集架构（[instruction set architecture](http://en.wikipedia.org/wiki/Instruction_set)，ISA）的指令序列，中间转换为目标ISA的指令序列并加以执行，输出为程序的执行结果的，就是VM。**源与目标ISA可以是同一种，这是所谓same-ISA VM。 



如果采用编译方式，VM会把输入的指令先转换为某种能被底下的系统直接执行的形式（一般就是native code），然后再执行之；如果采用解释方式，则VM会把输入的指令逐条直接执行。 
换个角度说，我觉得采用编译和解释方式实现虚拟机最大的区别就在于是否存下目标代码：**编译的话会把输入的源程序以某种单位（例如[基本块](http://en.wikipedia.org/wiki/Basic_block)/函数/方法/trace等）翻译生成为目标代码，并存下来（无论是存在内存中还是磁盘上，无所谓），后续执行可以复用之；解释的话则是把源程序中的指令逐条解释，不生成也不存下目标代码，后续执行没有多少可复用的信息。**



有些稍微先进一点的解释器**可能会优化输入的源程序**，把满足某些模式的指令序列合并为“超级指令”；



4.一二三地址指令

用C的语法来写这么一个语句： 
C代码  

```C
a = b + c;
```

如果把它变成这种形式： 

```
add a, b, c
```

那看起来就更像机器指令了，对吧？这种就是所谓“**三地址指令**”（3-address instruction），一般形式为op dest, src1, src2；



C里要是这样写的话： 
C代码  

```C
a += b;  
```

变成: 

```
add a, b 
```


这就是所谓“**二地址指令**”，一般形式为： op dest, src 
它要支持二元操作，就只能把**其中一个源同时也作为目标。上面的add a, b在执行过后，就会破坏a原有的值，而b的值保持不变**。x86系列的处理器就是二地址形式的。 



显然，指令集可以是任意“n地址”的，n属于自然数。那么一地址形式的指令集是怎样的呢？ 
想像一下这样一组指令序列： 
add 5 
sub 3 
这只指定了操作的源，那目标是什么？一般来说，这种运算的目标是被称为“**累加器”（accumulator）的专用寄存器，所有运算都靠更新累加器的状态来完成。**那么上面两条指令用C来写就类似： 

```c
acc += 5;  
acc -= 3;  
```

#### 那“n地址”的n如果是0的话呢？ 

```java
iconst_1  
iconst_2  
iadd  
istore_0  
```

零地址意味着源与目标都是隐含参数，其实现依赖于一种常见的数据结构——没错，就是栈。上面的iconst_1、iconst_2两条指令，分别向一个叫做“求值栈”（evaluation stack，也叫做operand stack“操作数栈”或者expression stack“表达式栈”）的地方压入整型常量1、2。

iadd指令则从求值栈顶弹出2个值，将值相加，然后把结果压回到栈顶。istore_0指令从求值栈顶弹出一个值，并将值保存到局部变量区的第一个位置（slot 0）。 
零地址形式的指令集一般就是通过“基于栈的架构”来实现的。请一定要注意，这个栈是指“求值栈”，而不是与系统调用栈（system call stack，或者就叫system stack）。千万别弄混了。有些虚拟机把求值栈实现在系统调用栈上，但两者概念上不是一个东西。 



由于指令的源与目标都是隐含的，**零地址指令的“密度”可以非常高——可以用更少空间放下更多条指令。因此在空间紧缺的环境中，零地址指令是种可取的设计。**但零地址指令要完成一件事情，一般会比二地址或者三地址指令许多更多条指令。

### 4的重点

**<u>假如一个VM采用基于寄存器的架构</u>**（它接受的指令集大概就是二地址或者三地址形式的），为了高效执行，一般会希望能把源架构中的寄存器映射到实际机器上寄存器上。但是VM里有些很重要的辅助数据会经常被访问，例如一些VM会保存源指令序列的程序计数器（program counter，PC），为了效率，这些数据也得放在实际机器的寄存器里。**如果源架构中寄存器的数量跟实际机器的一样，或者前者比后者更多，那源架构的寄存器就没办法都映射到实际机器的寄存器上**；这样VM实现起来比较麻烦，与能够全部映射相比效率也会大打折扣。



**<u>如果一个VM采用基于栈的架构</u>**，则无论在怎样的实际机器上，都很好实现——它的源架构里没有任何通用寄存器，所以实现VM时可以比较自由的分配实际机器的寄存器。于是这样的VM可移植性就比较高。作为优化，基于栈的VM可以用编译方式实现，“求值栈”实际上也可以由编译器映射到寄存器上，减轻数据移动的开销。

### IO开销

一般认为基于寄存器的架构对VM来说也是更快的，原因是：虽然零地址指令更紧凑，但完成操作需要更多的load/store指令，也意味着更多的指令分派（instruction dispatch）次数与内存访问次数；**访问内存是执行速度的一个重要瓶颈，二地址或三地址指令虽然每条指令占的空间较多，但总体来说可以用更少的指令完成操作，指令分派与内存访问次数都较少。** 



回到PHP

虚拟机是一种抽象的计算机，是对真实计算机的虚拟和模拟，现在的计算机有不同的 指令集架构([ISA: Instruction Set Architecture](http://homedir.jct.ac.il/~citron/ca/isa.html))， ISA是处理器的一个部分，不同的处理器会有不同的架构，最常见的有3种：

- 基于栈的[Stack Machines](http://en.wikipedia.org/wiki/Stack_machine): 操作数保存在栈上。 而不是使用寄存器来保存，现在很少有真实机器采用这个模型。对于虚拟机来说因为指令空间占用少， 并且实现简单，很多虚拟机采用这种模型，比如：JVM，HHVM等。
- 基于累加器的Accumulator Machines。这个模型使用称作累加器(Accumulator)的的寄存器来保存 一个操作数以及操作的结果
- 基于通用寄存器的General-Purpose-Register Machines，这些寄存器没有特殊的用途。 编译器可以将操作数保存在这些寄存器中。ZendVM采用的就是基于寄存器的架构。



\```php <?php

$a = 0;

if ($a > 0) { echo $a; } else { echo 4; } ```

下图为上面PHP代码编译后的opcode，可以看出来操作数使用的是一些数字，他们也可以理解对应于 物理机的寄存器，不同的是在这一个层次寄存器的数量可以理解为无限的，而物理机的寄存器是有限的。

\```

## line # * op fetch ext return operands

4 0 > ASSIGN !0, 0 6 1 IS_SMALLER ~1 0, !0 2 > JMPZ ~1, ->5 7 3 > ECHO !0 8 4 > JMP ->6 9 5 > ECHO 4 10 6 > > RETURN 1 ```

我们再看看基于栈的HHVM生成的opcode

```
Pseudo-main at 0 (ID 0) // line 4 0: Int 0 9: SetL 0 11: PopC // line 6 12: Int 0 21: CGetL2 0 23: Gt 24: JmpZ 14 (38) // line 7 29: CGetL 0 31: Print 32: PopC // line 10 33: Jmp 16 (49) // line 9 38: Int 4 47: Print 48: PopC 49: Int 1 58: RetC Pseudo-main at 0 (ID 0)
```

它就是基于栈的模式，指令的操作数都是保存在栈上的。可以看出来，相比于Zend的实现， 指令数量多了不少。**虽然指令数多了不少，在实际项目中，由于HHVM使用了JIT技术， 这些指令并不会解释执行，所以HHVM会比PHP快不少**。



再回到Zend

Zend引擎的核心文件都在$PHP_SRC/Zend/目录下面。不过最为核心的文件只有如下几个：

1. PHP语法实现

   - Zend/zend_language_scanner.l
   - Zend/zend_language_parser.y

2. Opcode编译

   - Zend/zend_compile.c

3. 执行引擎

   - Zend/zend_vm_*
   - Zend/zend_execute.

   ​

   ## 中间代码层

   当Zend虚拟机执行一个PHP代码时，它需要内存来存储许多东西， 比如，中间代码，PHP自带的函数列表，用户定义的函数列表，PHP自带的类，用户自定义的类， 常量，程序创建的对象，传递给函数或方法的参数，返回值，局部变量以及一些运算的中间结果等。 我们把这些所有的存放数据的地方称为中间数据层。

   如果PHP以mod扩展的方式依附于Apache2服务器运行，中间数据层的部分数据可能会被多个线程共享，比如PHP自带的函数列表等。 如果只考虑单个进程的方式，当一个进程被创建时它就会被加载PHP自带的各种函数列表，类列表，常量列表等。 当解释层将PHP代码编译完成后，各种用户自定义的函数，类或常量会添加到之前的列表中， 只是这些函数在其自身的结构中某些字段的赋值是不一样的。

   **当执行引擎执行生成的中间代码时，会在Zend虚拟机的栈中添加一个新的执行中间数据结构（zend_execute_data）， 它包括当前执行过程的活动符号列表的快照、一些局部变量等。**