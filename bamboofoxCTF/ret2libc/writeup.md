## 要用題目附的 libc.so.6

<img width="509" height="283" alt="image" src="https://github.com/user-attachments/assets/64dc38cd-3847-418a-9402-947b743f7ce2" />

## 執行後會直接給你 puts 函數的位置
利用 recv() 接收下來後是 bytes
利用 decode() 將 bytes 轉為 str

<img width="576" height="113" alt="image" src="https://github.com/user-attachments/assets/efbb0526-1d87-42d8-8d07-f968b29ebe37" />

## 計算 libc base 然後算出 system 函數位置

<img width="806" height="200" alt="image" src="https://github.com/user-attachments/assets/1aafbc81-eff4-4834-a294-56d03f74dfa3" />

算出來後用 BOF 覆蓋 return address 以及 parameter
