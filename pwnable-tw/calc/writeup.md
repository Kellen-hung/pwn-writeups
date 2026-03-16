### source code 

經由 IDA 反組譯後, 有四個主要的 functions, main -> calc() -> get_expr() -> parse_expr() -> eval() -> print result

### **有開 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**

看 source code 沒有什麼想法, 我覺得計算機本來就很複雜, 得慢慢 trace code 來理解

main 先呼叫 calc()
calc() 再呼叫 get_expr() 進行輸入
get_expr() 沒什麼特別的, 只接收 0~9 + - * / %
parse_expr() 是重點, 一開始測試程式的時候發現 -12+34 這種第一個字為 operator 的計算會出現問題, 就是在這裡發生的

pool[101] 宣告在 calc() 裡, 專門用來存計算過程的數字, 其中 pool[0] 用來當作計算的 base address

一開始用 -8+4 這種 case 會導致 pool[-7] = 4, 這樣很難覆寫到 return address, 所以用 +8+4 來 trace, 會發現 pool[9] = 4, 也就是在 +num1+num2 的 case 下, pool[num1+1] = num2, 也就是可以任意寫入 pool 後的位置, 竄改在 calc 裡的 return address


### solution

直接用 GDB 暴力計算 pool 跟 ret_addr 的距離再除以4 後再減1, 就是 num1 要填的值 (畢竟 ASLR 不影響他們的相對位置), 另外 num2 就是填那位置要寫入的值

因為這個 ELF 是 static linked, 所以接著構造 ROP chain 呼叫 execve("/bin/sh",0,0) 就結束了

要從 ROP chain 最後面開始寫入, 例如從 370 寫到 361, 因為如果先寫 361 再寫 362 的話 361 會被蓋掉, 沒有去 trace 為何, 觀察出來的 (我懶)

我用最土炮的方式寫出 ROP chain, 一定有更簡單的方法