## 是 ret2libc 題目
這題沒有給任何在 libc 裡的函數的 address，因此需自己找出 puts 函數的位址

## 題目一開始有限制 max_len = 20 bytes
因為會導致input不夠用，所以利用 BOF 將 max_len 覆蓋成 0xff
![image](https://hackmd.io/_uploads/rJ5K_KlN1l.png)

## 接下來要取得 puts 函數的 address 並且返回 main 原點重新開始 (ROP)
在 main 函數 ret 回 puts@plt (搭配參數 puts@got.plt)
-> 會導向 puts 函數並輸出 puts 函數的真正 address
-> puts(puts@got.plt)

### 知識點

**第一次呼叫：延遲綁定（Lazy Binding）**
當調用外部函數（如 puts）時，先進 puts@plt 然後再跳進 puts@got.plt，但這時 puts 尚未載入到 GOT，GOT 中的 puts 條目指向一段 PLT 初始化代碼，這段代碼會觸發動態鏈接器（ld.so）進行符號解析，將 puts address 載入到 GOT 中的相應位置

**非第一次呼叫：直接跳轉**
此時 GOT 中已儲存 puts 函數的 address，puts@plt 直接跳轉到 puts@got.plt 將 puts address 直接取出來使用

-> 因此 puts(puts@got.plt) 會將 puts address leak 出來

![image](https://hackmd.io/_uploads/S1PQ-cgNyl.png)

## 跑第二輪 main function
進行 ret2libc，將第一個 buf 填入 /bin/sh 後 call system
![image](https://hackmd.io/_uploads/HJBfGcxNye.png)

## 問題
我用自己的電腦開伺服器 leak 不出 puts 的 address，但連上題目指定的 ip port 就可以，找不出原因