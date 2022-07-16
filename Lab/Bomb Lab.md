# Bomb Lab

---

 Bomb Lab食用方法: 

- 开始前, 学习gdb的基本操作
- 使用gdb而非objdump完成该lab(会方便很多)
- 此Lab需要第3章部分内容。3.9足够完成前6个lab(看懂Canary影响不大, 后面的浮点数指令则完全不需要)

---

## <span id = "index">目录</span>

[Bomb.c](#id = bomb.c)

**Phases**

​		[phase_1](#phase_1)		[phase_2](#phase_2)		[phase_3](#phase_3)

​		[phase_4](#phase_4)		[phase_5](#phase_5)		[phase_6](#phase_6)

​		[secret_phase](#secret_phase)

[**总结**](#总结)

---

### <span id = "bomb.c">[Bomb.c](#Index)</span>

​		仔细阅读此文件，可以大概明白实验内容和要求。例如有6个phase, 还有个奇怪的**phase_defused**函数。

---

### <span id = "phase_1">[phase_1](#index)</span>

答案: Border relations with Canada have never been better.

​		我们需要先学习一些gdb的基本操作，

- gdb bomb     使用gdb打开bomb
- run (<file\>)    运行bomb. 若file不为空, 则使用其运行bomb.
- b *address   or   b <function_name\>    设置断点
- delete (i)       删除全部断点(断点i)
- continue        运行到下个断点
- finish             运行到当前函数结束
- p *(<form\>\*) address    打印指针
- x/s  address 打印某一地点的string
- stepi (n)        运行下1(n)步
- x/<number\> address 打印某一地点的number条信息
- disas <function_name\> 反汇编函数
- info r / f         显示所有寄存器 / 栈帧的信息

​		首先, 我们看一下函数的汇编代码。代码中用到了strings_not_equal函数。函数很长，但可猜测是判断两个string是否相等，有两个参数，返回类型为int。如果两个string不相同，则explode。

> ​		书上提到，函数前两个参数所在的寄存器是rdi和rsi，而此处mov $0x402400 %rsi，因此猜测是两个参数的函数；返回值在rax中，下面进行条件判断使用eax，猜测返回类型是int. 

​		我们使用x/s 0x402400看一下该位置的字符串, 显示为"Border relations with Canada have never been better."。输入后进入phase_2，猜测正确。

![image-20220716173424534](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716173424534.png)

---

### <span id = "phase_2">[phase_2](#index)</span>

答案: 1 2 4 8 16 32

​		还是先看汇编代码。此题难度明显高于上一题，但耐心分析会发现只是简单循环。phase_2使用了read_six_numbers函数，其中+46使用了一个sscanf@plt，这是一个信号函数，返回值是输入值的个数，分析可知要输入6个数存入栈中，如果不足则会explode。

> 要注意一点，如果输入的数超过6个，实际上并不会explode，而是放弃读取多余的数。

- +18出现第一个条件跳转，说明输入的第1个数必须为1，否则explode
- 分析可知，之后每个数是上一个数的2倍，可得答案

![image-20220716174242564](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716174242564.png)

---

### <span id = "phase_3">[phase_3](#index)</span>

答案：0 207(不唯一)

​		汇编代码如下。此题考查的是跳转表(switch语句), 看懂了就很容易。

- +14 这个是sscanf输入的内容, 使用x/s 0x4025cf可以看到需要输入两个十进制整数。
- +39 输入的第一个数不能＞7
- 然后进入跳转表，可以在explode处设一个断点，随便输入两个数，rax会获得跳转表中相应的值，第二次再做更改即可。
- 答案不唯一，有7组解。

![image-20220716175556710](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716175556710.png)

---

### <span id = "phase_4">[phase_4](#index)</span>

​		答案: 1 0 DrEvil(不唯一)(隐藏关钥匙)

​		本来写了过程，没保存，就直接贴翻译的代码了。这题要分析每一行，然后写出递归式计算结果。

- +34 有一个限制条件，所以可以直接用代码跑出来，懒得算了。活用现代化工具也是个好方法嘛:D

```c
#include<stdio.h>
#include<stdlib.h>
int eax, edi, esi, edx, ecx;

void phase_4();

void func4(int eax, int edi, int esi, int edx, int ecx)
{
	eax = edx - esi;
	ecx  = eax >> 31;
	eax = (eax + ecx) >> 1;
	ecx = esi + eax;
	if(ecx <= edi){
		eax = 0;
		if(ecx >= edi){
			;
		}else{
			esi = ecx + 1;
			func4(eax, edi, esi, edx, ecx); 
		}
	}else{
		edx = ecx - 1;
		func4(eax, edi, esi, edx, ecx);
		eax = 2 * eax;
	}
}

void explode()
{
	printf("Bomb explodes!\n");
	exit(0);
}

void phase_4()
{
	scanf("%d %d", &edx, &ecx);
	if(edx > 14)
		explode();
	edx = 0xe;
	esi = 0x0;
	edi = edx;
	func4(eax, edi, esi, edx, ecx);
	if(eax)
		explode();
	if(ecx != 0)
		explode();
	printf("Go on...%d %d", edx, ecx);
}

int main(){
	phase_4();
	return 0;
}
```



---

### <span id = "phase_5">[phase_5](#index)</span>

答案: 9on567

​		汇编代码如下。这题出现了Canary，但完全没看出有什么作用。

- +24 - 32 输入长度为6的string

- +41 - 52 rdx = (%rbx, %rax, 1) & 0xf

- +55 - 74 0x4024b0储存的string是一堆无意义字符，每次从中获得0x4024b0(%rdx)放入0x10(%rsp, %rax, 1)，进行6次

- 完成上述过程后，将6次获得的字符组成的string与0x40245e的string比较，相同即通过

  > 简单来说，我们要输入6个字符c1-6，&0xf后得index1-6，然后使0x4024b0(index) == 0x40245e(index)即可。这两个string可以通过x/s查看。

![image-20220716181626620](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716181626620.png)

---

### <span id = "phase_6">[phase_6](#index)</span>

答案: 4 3 2 1 6 5

[思路](#i6)

- <span id = "as6">汇编代码如下</span>, 按照惯例进行分析

  ```assembly
     0x00000000004010f4 <+0>:		push   %r14
     0x00000000004010f6 <+2>:		push   %r13
     0x00000000004010f8 <+4>:		push   %r12
     0x00000000004010fa <+6>:		push   %rbp
     0x00000000004010fb <+7>:		push   %rbx
     0x00000000004010fc <+8>:		sub    $0x50,%rsp
     0x0000000000401100 <+12>:	mov    %rsp,%r13
     0x0000000000401103 <+15>:	mov    %rsp,%rsi
     0x0000000000401106 <+18>:	callq  0x40145c <read_six_numbers>
     0x000000000040110b <+23>:	mov    %rsp,%r14
     0x000000000040110e <+26>:	mov    $0x0,%r12d
     0x0000000000401114 <+32>:	mov    %r13,%rbp
     0x0000000000401117 <+35>:	mov    0x0(%r13),%eax
     0x000000000040111b <+39>:	sub    $0x1,%eax
     0x000000000040111e <+42>:	cmp    $0x5,%eax
     0x0000000000401121 <+45>:	jbe    0x401128 <phase_6+52>
     0x0000000000401123 <+47>:	callq  0x40143a <explode_bomb>
     0x0000000000401128 <+52>:	add    $0x1,%r12d
     0x000000000040112c <+56>:	cmp    $0x6,%r12d
     0x0000000000401130 <+60>:	je     0x401153 <phase_6+95>
     0x0000000000401132 <+62>:	mov    %r12d,%ebx
     0x0000000000401135 <+65>:	movslq %ebx,%rax
     0x0000000000401138 <+68>:	mov    (%rsp,%rax,4),%eax
     0x000000000040113b <+71>:	cmp    %eax,0x0(%rbp)
     0x000000000040113e <+74>:	jne    0x401145 <phase_6+81>
     0x0000000000401140 <+76>:	callq  0x40143a <explode_bomb>
     0x0000000000401145 <+81>:	add    $0x1,%ebx
     0x0000000000401148 <+84>:	cmp    $0x5,%ebx
     0x000000000040114b <+87>:	jle    0x401135 <phase_6+65>
     0x000000000040114d <+89>:	add    $0x4,%r13
     0x0000000000401151 <+93>:	jmp    0x401114 <phase_6+32>
     0x0000000000401153 <+95>:	lea    0x18(%rsp),%rsi
     0x0000000000401158 <+100>:	mov    %r14,%rax
     0x000000000040115b <+103>:	mov    $0x7,%ecx
     0x0000000000401160 <+108>:	mov    %ecx,%edx
     0x0000000000401162 <+110>:	sub    (%rax),%edx
     0x0000000000401164 <+112>:	mov    %edx,(%rax)
     0x0000000000401166 <+114>:	add    $0x4,%rax
     0x000000000040116a <+118>:	cmp    %rsi,%rax
     0x000000000040116d <+121>:	jne    0x401160 <phase_6+108>
     0x000000000040116f <+123>:	mov    $0x0,%esi
     0x0000000000401174 <+128>:	jmp    0x401197 <phase_6+163>
     0x0000000000401176 <+130>:	mov    0x8(%rdx),%rdx
     0x000000000040117a <+134>:	add    $0x1,%eax
     0x000000000040117d <+137>:	cmp    %ecx,%eax
     0x000000000040117f <+139>:	jne    0x401176 <phase_6+130>
     0x0000000000401181 <+141>:	jmp    0x401188 <phase_6+148>
     0x0000000000401183 <+143>:	mov    $0x6032d0,%edx
     0x0000000000401188 <+148>:	mov    %rdx,0x20(%rsp,%rsi,2)
     0x000000000040118d <+153>:	add    $0x4,%rsi
     0x0000000000401191 <+157>:	cmp    $0x18,%rsi
     0x0000000000401195 <+161>:	je     0x4011ab <phase_6+183>
     0x0000000000401197 <+163>:	mov    (%rsp,%rsi,1),%ecx
     0x000000000040119a <+166>:	cmp    $0x1,%ecx
     0x000000000040119d <+169>:	jle    0x401183 <phase_6+143>
     0x000000000040119f <+171>:	mov    $0x1,%eax
     0x00000000004011a4 <+176>:	mov    $0x6032d0,%edx
     0x00000000004011a9 <+181>:	jmp    0x401176 <phase_6+130>
     0x00000000004011ab <+183>:	mov    0x20(%rsp),%rbx
     0x00000000004011b0 <+188>:	lea    0x28(%rsp),%rax
     0x00000000004011b5 <+193>:	lea    0x50(%rsp),%rsi
     0x00000000004011ba <+198>:	mov    %rbx,%rcx
     0x00000000004011bd <+201>:	mov    (%rax),%rdx
     0x00000000004011c0 <+204>:	mov    %rdx,0x8(%rcx)
     0x00000000004011c4 <+208>:	add    $0x8,%rax
     0x00000000004011c8 <+212>:	cmp    %rsi,%rax
     0x00000000004011cb <+215>:	je     0x4011d2 <phase_6+222>
     0x00000000004011cd <+217>:	mov    %rdx,%rcx
     0x00000000004011d0 <+220>:	jmp    0x4011bd <phase_6+201> 
     0x00000000004011d2 <+222>:	movq   $0x0,0x8(%rdx)
     0x00000000004011da <+230>:	mov    $0x5,%ebp
     0x00000000004011df <+235>:	mov    0x8(%rbx),%rax
     0x00000000004011e3 <+239>:	mov    (%rax),%eax
     0x00000000004011e5 <+241>:	cmp    %eax,(%rbx)
     0x00000000004011e7 <+243>:	jge    0x4011ee <phase_6+250>
     0x00000000004011e9 <+245>:	callq  0x40143a <explode_bomb>
     0x00000000004011ee <+250>:	mov    0x8(%rbx),%rbx
     0x00000000004011f2 <+254>:	sub    $0x1,%ebp
     0x00000000004011f5 <+257>:	jne    0x4011df <phase_6+235>
     0x00000000004011f7 <+259>:	add    $0x50,%rsp
     0x00000000004011fb <+263>:	pop    %rbx
     0x00000000004011fc <+264>:	pop    %rbp
     0x00000000004011fd <+265>:	pop    %r12
     0x00000000004011ff <+267>:	pop    %r13
     0x0000000000401201 <+269>:	pop    %r14
     0x0000000000401203 <+271>:	retq 
  ```

- <span id = "i6">思路</span>

> ​	**观察反汇编代码, 可以发现代码分为五块, 分别是+0-93, +95-121, +123-181, +183-220 +223-257. 满足其中一块的条件后会进入下一块, 可以对这五块分别分析. **
>
> +0 - 93
>
> - 注意到+18出现了read_six_number函数, 在phase_2中了解到需要输入6个数字
>
> - +35-47 意为%r13指向的值减1,不大于5. 由+12知%r13指向栈顶, 其中的值是输入的第一个数字, 即第一个数字在1-6皆可
>
> - +26, +52-87 循环
>
>   > 初始化: r12d = 1; ebx = 1;
>   >
>   > 条件: 	ebx <= 5;
>   >
>   > update: ebx++;
>   >
>   > ​	循环体: eax = (%rsp, %rbx, 4) must > (%rbp);	//(%rbp)是栈顶的数，即剩下的5个数与栈顶的数不相等
>
>   
>
> - +89-93   将%r13加4, 回到+32, 共循环6次, 其中最后一次特殊. 每次循环
>
>   - %r13的位置上移4, 指向的值是输入的下一个数字，因此输入的六个数字都为1-6. 最后一次循环只检测这步
>   - +52开始的循环初始化值均+1, 可看出+52循环用于检测输入的6个数是否有重复. 如果重复, 则explode bomb. 
>
> **目前, 输入是1-6的一个排列**
>
> +95-121
>
> - +95-121   是一个循环, 分析代码可知, 这个循环将输入的6个数字n变为7-n
>
>   > 初始化: rsi = rsp + 24, rax = r14(r14 = rsp), ecx = 7;
>   >
>   > 条件:     rax != rsi
>   >
>   > update: rax += 4;
>   >
>   > ​	循环体: edx = ecx - *rax; *rax = edx;
>
> **第二块将输入数字n变为7-n**
>
> +123-181
>
> - +130   rdx = 0x8(rdx), 通过测试可知 0x8(rdx)  = rdx + 0x10;
>
> - x/24 0x6032d0, 可以看到好像是一个链表
>
>   ![image-20220716131239028](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716131239028.png)
>
>   > struct node{
>   >
>   > ​	int val_1;	//given
>   >
>   > ​	int val_2;	//7-n
>   >
>   > ​	node* next;
>   >
>   > }
>
> - 整段代码是一个二重循环, 伪代码如下(原有的if-else可以合并). 
>
>   > for(rsi = 0; rsi != 0x18; rsi+=4){
>   > 	ecx = (rsp, rsi, 1), eax = 1, edx = 0x6032d0;
>   >
>   > ​	while(eax != ecx){
>   > ​		rdx = rdx + 16;
>   >
>   > ​		eax ++;
>   >
>   > ​	}
>   >
>   > ​	0x20(rsp, rsi, 2) = rdx;
>   >
>   > }
>
> **第三块对第二块的第m+1个数n+1, 使0x20(rsp, m, 8) = 0x6032d0 + n \* 16**
>
> +183 - 220
>
> - 这一块非常难看懂...总之是把0x20(rsp)到0x48(rsp)的内容串起来, 顺序按内存位置递增
>
> +220-
>
> - 检测链表是否按照val_1降序排列 所以第二块的序列应该是3 4 5 6 1 2, 输入的数是4 3 2 1 6 5

---

### <span id = "secret_phase">[secret_phase](#index)</span>

答案: 22

[思路](#i7)

​	That's not the end. 如果之前使用过objdump, 会发现其中还有一个secret_phase, 函数入口在phase_defused中. 反汇编phase_defused函数, 如下图

​	![image-20220716151240688](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716151240688.png)

​		注意到secret_phase在+108, 而+27会跳转到+123. 调试知, num_input_strings返回已完成的phase的个数, 即仅当完成前6个phase, 才可以进入secret_phase. 

​		此外, 还要不满足+62和+81的跳转条件. 前者查看0x603870的内容知, 是phase_4的输入(当时要求的是输入为2, 但实际上大于2也只会读取前两个数据). 后者要求输入的string与$0x402622相同, 为DrEvil.

​		下面的0x4024f8和0x402520是进入phase的消息提示, 而非有效信息. 

![image-20220716153654165](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716153654165.png)

<span id = "i7">**思路**</span>(上为secret_phase)

> 首先输入多个数字进行尝试，发现只要输入一个数，存储在rax中. 
>
> +30, 输入的数≤1001, 之后进入fun7(rdi, rsi)
>
> ![image-20220716161654102](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716161654102.png)
>
> ​		fun7需要用到secret_phase中rdi的值0x6030f0, 如下图所示。看着很吓人，观察指针发现其实很友好，是一棵完全二叉树(而且是BST)，需要自己画一下。
>
> ​		此题可翻译为: 输入一个数，在该BST中搜索。如果未找到，返回负数(肯定explode); 否则，若向rchild前进过，先令rax=0。然后回溯，每次向rchild前进，rax=2*rax+1; 向lchild前进，rax = 2\*rax.
>
> ​		由于phase要求eax = 2, 因此应该为left-right-equal，即n32的值22
>
> ![image-20220716165029188](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716165029188.png)

---

<span id = "总结">[总结](#index)</span>: 学到了gdb的使用，增强了汇编代码的阅读能力，觉得是值得的，确实很有意思。

![image-20220716173141279](https://github.com/polaris-cyy/CSAPP/blob/main/picture/image-20220716173141279.png)
