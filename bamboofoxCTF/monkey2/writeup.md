## 目標: 將 system 裡的參數指到 /bin/sh
![image](https://hackmd.io/_uploads/Syd83Sg4Jg.png)

## 用 Format String 做記憶體寫入

先找出 choice (int) 在哪個位置
![image](https://hackmd.io/_uploads/HyohCSeVJx.png)

再利用這個 choice 去推算其他需要的 address
![image](https://hackmd.io/_uploads/Bkd9J8lVJg.png)

再來就是計算需要的字元數進行 %n 寫入
![image](https://hackmd.io/_uploads/BkZCJIeNye.png)

將 name_addr 分成 4bytes 方便計算
& 0xff 取出 1byte
\>\>8 前進下個 byte