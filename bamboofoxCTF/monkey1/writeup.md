**使用 Format String Exploit**

## 什麼是 Format String Exploit?
[說明連結](https://r888800009.github.io/software/security/binary/format-string-attack/)

例如

```bash=
char str[30];
scanf("%s",&str);
printf(str);      //沒用 printf("%s",str)
```
input 輸入 %p 可以顯示 stack 裡的資料

## 看 flag function ,得知需要將banana改為 0x3132000a

<img width="612" height="436" alt="image" src="https://github.com/user-attachments/assets/e68d1ad2-5a1d-4119-8ec7-6857e396fd52" />

## 使用 GDB 找 banana 位置
~~觀察在 main function 裡面跑 banana = 1 時的位置~~
~~-> **banana的位置在 0xffffd474**~~

待確認：因為 banana 在宣告時就被 assign 為 1，在 main 裡沒紀錄 banana 的位置，因此需要確認 choice 的位置然後 +4 才是 banana 的位置
找時間自己寫程式確認!!

<img width="922" height="327" alt="image" src="https://github.com/user-attachments/assets/cf647b43-14a8-4164-a09e-125b5304da9d" />

## 在 function program 裡可進行 Format String Exploit

裡面有
```bash=
printf(temp)
```
因為每次執行程式 banana 的位置都會不同，因此第一次輸入要先 leak choice 的位置並 recv 下來，然後進行 printf 的操作，先利用輸入
```
AAAABBBBCCCC_%p.%p.%p.%p.%p.%p ...
```
或者
```
AAAABBBBCCCC
```
後利用 GDB 的 x/40wx $esp 進行觀察 temp 是第幾個參數
-> **此題 temp 為 esp 後的第七個參數($7)**

## 構建 payload
<img width="398" height="155" alt="image" src="https://github.com/user-attachments/assets/abf906dc-d46d-4ec7-b738-030a76b2ef1e" />

意思為: 將 banana addr 以及 banana_addr + 2 寫進 temp 裡，此時 banana_addr 為 $7，banana_addr + 2 為 $8，利用 %c 以及 %hn 的組合進行記憶體寫入

%10000c 為輸出 10000 個字元
%n 會將前面字元個數寫進後面地址裡
%n 寫進 4 bytes
%hn 寫進 2 bytes
%hhn 寫進 1 bytes
