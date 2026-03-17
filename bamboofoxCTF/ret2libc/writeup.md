## 要用題目附的 libc.so.6
![image](https://hackmd.io/_uploads/BJloV8lEJg.png)

## 執行後會直接給你 puts 函數的位置
利用 recv() 接收下來後是 bytes
利用 decode() 將 bytes 轉為 str
![image](https://hackmd.io/_uploads/BJGJSUlNJl.png)

## 計算 libc base 然後算出 system 函數位置
![image](https://hackmd.io/_uploads/HkE_eDeVyl.png)
算出來後用 BOF 覆蓋 return address 以及 parameter