---
title: 熵與資訊增益：從資訊論到機器學習
published: 2026-04-11
description: "深入探討香農熵 (Shannon Entropy) 的定義與其在資訊增益 (Information Gain) 中的應用"
pinned: false
tags: [entropy, information-gain, machine-learning, information-theory]
category: Computer Science
draft: true
---

## 導引：資訊是什麼？

在日常生活中，我們經常提到「這條資訊對我很重要」，從辭典的定義上來看可以有兩種解釋，可以從這些解釋推斷資訊就是人類定義的資料且本身必須要有意義。

1. 電腦上指對使用者有用之資料和訊息的總稱。以別於未經處理過的資料。
2. 泛指一般資料和訊息。

但在資訊理論中，資訊 (Information) 有一個非常具體且數學化的定義：

> 
 Abstractly, in this approach information can be thought of as the resolution of uncertainty.[2]
 >
 **資訊是用來消除不確定性的量度。**

想像一下兩個場景：
1.  **拋一枚公正的硬幣：** 你在拋之前，對結果（正面或反面）的不確定性相對較小，因為只有兩種可能。
2.  **擲一顆公正的六面骰子：** 你在擲之前，對結果的不確定性較大，因為有六種等機率的可能性。

直覺上，告訴你骰子的結果「包含的資訊量」比告訴你硬幣的結果要多。這引出了一個核心問題：我們如何從第一性原理出發，定義一個科學的、能量化這種「不確定性」的度量標尺？

## 從第一性原理視角解釋設計衡量資訊的指標
首先先來想像，如果要設計一個指標用來衡量資訊，那麼會需要哪些邏輯。

### 1. 越罕見資訊越多
資訊的本質是為了消除不確定性，如果我說出『太陽明天會從東邊升起』，那麼你會覺得我在說廢話，因為這事情發生的機率高，所以資訊量很低；那如果我說出『明天會有一場流星雨』（機率低），資訊量就會變高。

所以我們會需要一個函數$f(p)$，當$p$機率越低這個函數所返回的結果會越大。

### 2. 獨立事件的資訊應該要可以『相加』
假設今天有兩個獨立事件，『今天在下雨』，『我抽到一個獎品』。理所當然這兩個獨立事件的所夾帶的資訊應該要是兩者的總和。

但是在機率論中，兩個獨立事件同時發生的機率是$P(A) \times P(B)$。

這就是我們面臨的挑戰：我們需要一個函數 $f$，能把「機率的相乘」映射到「資訊的相加」。

在數學的世界裡，具備這種特性的函數極其稀有，它就是 對數函數 (Logarithm)。
$$ \text{因為 } \log(P(A) \cdot P(B)) = \log P(A) + \log P(B) $$

為了讓結果在 $p$ 很小時（即 $\log p$
為負數時）呈現正值，我們順理成章地在前面加上一個負號。至此，我們不是「學會」了公式，而是「推導」出了資訊量的唯
一可能形式：
$$ I(p) = -\log p $$

### 為什麼是 $-\log(p)$？

在數學上，唯一能將「相乘關係（機率）」轉換為「相加關係（資訊量）」且滿足單調遞減的函數，就是**對數函數的負值**。

因此，我們定義一個機率為 $p$ 的單一事件所攜帶的資訊量（又稱 **自資訊 Self-information** 或 **驚訝度 Surprisal**）為：
$$I(p) = -\log_2 p$$

（註：當對數的底數為 2 時，資訊量的單位是 **位元 (bits)**。）

## 熵 (Entropy) 的定義：資訊量的期望值

有了「單一事件資訊量」的概念後，**熵 (Entropy)** 的本質就呼之欲出了：
**熵，就是一個隨機變數所有可能結果所攜帶資訊量的「平均值（期望值）」。**

如果一個隨機變數 $X$ 有 $n$ 種可能的結果 $\{x_1, x_2, \dots, x_n\}$，且每種結果發生的機率為 $p(x_i)$，則其熵 $H(X)$ 的定義為：

$$H(X) = E[I(p(x))] = -\sum_{i=1}^{n} p(x_i) \log_2 p(x_i)$$

### 例子分析

*   **完全確定：** 如果硬幣兩面都是正面（$p=1$），則 $H(X) = -(1 \log_2 1) = 0$。沒有不確定性，就沒有熵，因為結果早在預料之中。
*   **最大隨機：** 當所有結果機率相等時（均勻分佈），熵達到最大值。這就是為什麼一個混亂的系統（高度不確定）比一個有序的系統擁有更高的熵。


## 資訊增益 (Information Gain)

(內容待補充)

## 參考文獻 (References)

1. [教育部辭典"資訊"定義](dict.revised.moe.edu.tw/dictView.jsp?ID=137246&q=1&word=資訊)

2. [Information theory wikipedia](https://en.wikipedia.org/wiki/Information_theory)

- **Shannon, C. E. (1948).** "A Mathematical Theory of Communication." *Bell System Technical Journal*.
- **Cover, T. M., & Thomas, J. A. (2006).** *Elements of Information Theory* (2nd ed.). Wiley-Interscience.
- **Quinlan, J. R. (1986).** "Induction of Decision Trees." *Machine Learning*.
- **Quinlan, J. R. (1993).** *C4.5: Programs for Machine Learning*. Morgan Kaufmann.
- **Bishop, C. M. (2006).** *Pattern Recognition and Machine Learning*. Springer.
- **Murphy, K. P. (2012).** *Machine Learning: A Probabilistic Perspective*. MIT Press.
