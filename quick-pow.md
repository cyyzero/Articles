---
title: 快速幂
date: 2017-07-07 23:08:01
tags:
- 算法
- 位运算 
- 递归
catagories:
- 算法
---
来自《数据结构与算法分析》第二章。

书上介绍的是第二种**递归**的算法。之前还见过用**位运算**的方法，所以特意记录一下。这两种算法都是O(logn)。

- **递归版本**

	```
	int quickpow(int base, unsigned exp)
	{
		if(exp == 0)
			return 1;
		if(exp == 1)
			return base;
		if(exp % 2)
			return quickpow(base*base, exp/2)*base;
		else
			return quickpow(base*base, exp/2);
	}
	```

	思路：
	$$
	X^{n}=
	\begin{cases}
	1&\mbox{，n=0}\\\
	X&\mbox{，n=1}\\\
	{(X^{2})}^{n/2}&\mbox{，n为偶数}\\\
	X·{(X^{2})}^{(n-1)/2} &\mbox{，n为奇数}
	\end{cases}
	$$
	
---
- **位运算版本**

	```
	int quickpow(int base, unsigned exp)
	{
		int ans = 1;
		while(exp)
		{
			if(exp&1)
				ans *= base;
			base *= base;
			exp >>= 1;
		}
		return ans;
	}
	```

	思路：
	先举个例子。当底数为x，指数为5时， $x^{5}$ = $x^{4}$·x。5的二进制为0101，而4=$2^{2}$, 1=$2^{0}$。是不是看出了一点什么？
	再来理解这一行

	>base*=base;

	目的是将base不断累乘，以实现base再每次循环中分别等于$x^{1}$, $x^{2}$, $x^{4}$, $x^{8}$, $x^{16}$……

	第n次循环中判断exp右数第n位是否为1，如果是的话就将ans乘以base。循环最后将exp右移一位，以便下次循环判断对应bit。

	再回到那个例子。第一次循环，0101右数第一位是1，ans乘上了x。base累乘为$x^{2}$。0101右移一位变成010。第二次循环，对应bit为0，base累乘到$x^{4}$，0101右移成01。第三次循环，对应bit为1，ans再乘上base，既$x^{4}$。最后一次循环也同理。 最后得到的ans为$x^{5}$。