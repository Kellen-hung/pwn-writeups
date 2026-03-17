## 目標: 將 system 裡的參數指到 /bin/sh
<img width="470" height="139" alt="image" src="https://github.com/user-attachments/assets/446a6b97-611a-4653-9321-6f0e2b0ebbcc" />

## 用 Format String 做記憶體寫入

先找出 choice (int) 在哪個位置

<img width="494" height="121" alt="image" src="https://github.com/user-attachments/assets/7c85147a-de24-4b8c-85bc-85853f6adad8" />

再利用這個 choice 去推算其他需要的 address

<img width="535" height="89" alt="image" src="https://github.com/user-attachments/assets/cc8c0187-e92f-42a4-bdb1-6241b7e01834" />

再來就是計算需要的字元數進行 %n 寫入

<img width="932" height="364" alt="image" src="https://github.com/user-attachments/assets/970bdc8a-389a-447e-b437-73d81e14d876" />

將 name_addr 分成 4bytes 方便計算
& 0xff 取出 1byte
\>\>8 前進下個 byte
