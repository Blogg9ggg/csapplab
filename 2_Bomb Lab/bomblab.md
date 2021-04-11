1. 查看 `bomb.c`，大意就是说要通过 6 个阶段的 “考验“ 才能顺利拆除炸弹的雷管，否则它就会爆炸。

   

2. 阶段 1

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase1.png" style="zoom:80%;" /> 

   阶段 1 需要输入的是 `Border relations with Canada have never been better.`

   

3. 阶段 2

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase2.png" style="zoom: 67%;" /> 

   阶段 2 需要输入的是 `1 2 4 8 16 32`

   

4. 阶段 3

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase3.png" style="zoom: 67%;" />  

   要求输入 v6 和 v7，然后 result 会根据 v6 来赋值，要求 result == v7。我选择 `0 207`。

   

5. 阶段 4

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase4_1.png" style="zoom: 67%;" />  

   要求输入 v6 和 v7，并且 v7 等于 0，而 v6 输入函数 func4() 后得到的结果值也为 0。

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase4_2.png" style="zoom:67%;" />  

   `num1 = 0, num2 = 14`，这是一个递归函数，但只要稍加分析就能发现只要令 `v6 = 7` 即可返回 0，不用递归。

   

6. 阶段 5

   <img src="C:\电子图书馆\我的坚果云\CTF\CSAPP_lab\bomb\writeup\assets\phase5.png" style="zoom:67%;" />  

   输入 `ionefg` 即可。

   

7. 阶段 6

   这种情况下直接看代码就不如去 peda 一点一点动调。这个阶段其实就是在做如下几件事：

   1. 输入 6 个互不相同的数字，$a_1, a_2, a_3, a_4, a_5, a_6$；

   2. 求 $b_i = 7 - a_i, i = 1,2,3,4,5,6$；

   3. 求 $c_i = F(b_i), i = 1,2,3,4,5,6$。其中
      $$
      F(x) =  
      \begin{equation}
      	\begin{cases}
      		0x14c,x=1\\
      		0xa8, x=2\\
      		0x39c, x=3\\
      		0x2b3, x=4\\
      		0x1dd, x=5\\
      		0x1bb, x=6
      	\end{cases}
      \end{equation}
      $$

   4. 要求 $c_i$ 数组单调递减，最终的输入为 `4 3 2 1 6 5`

