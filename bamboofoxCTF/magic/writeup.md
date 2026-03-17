第14行+6為了跳過這三行 (0d是CR字元)
![image](https://hackmd.io/_uploads/ryoCayc7yl.png)


架構圖
![image](https://hackmd.io/_uploads/ryNTpk5mkg.png)


## 知識點
1. strlen()遇到 \x00 會停
2. scanf()遇到回車，換行會停
3. 只要在injection input前多加 \x00 不被strlen()讀進去就不會在do_magic function被洗亂