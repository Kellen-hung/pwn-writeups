### source code 
```
太多了懶得貼, 可以知道的是大多是沒用的地方(沒洞可打)
```

### **沒 canary 有開 NX 沒 FULL RELRO**
1. 在選單的 choice 可以 out of bound
2. me 的 struct 長這樣
    ```c=
    struct mes
    {
      int seed;
      int size;
      int position;
      int stone;
      int hp;
      int score;
      int data[76];
      char *host;
      int data2;
      char name_addr[128];
      int data3;
      int *funcs[11];
    };
    ```
3. 調用選單中的 function 的時候是呼叫 me->funcs[choice]
4. 所以如果 choice 可以 oob, 意味著也可以調用 name
5. 如果 name 可以修改, 那就達到任意位置執行

### solution - ret2libc
1. 雖然題目沒有給 libc, 但還是可以用 libc database 來知道他用什麼版本
2. 如果直接將 name 改成 puts@plt 的位置, 因 [esp] 不是 local variable, 所以不可控
3. 所以要將 esp 加上去, 讓他達到可控位置
4. 只要參數位置可控, 就可利用 ROP leak libc base
5. 最後再回到 main, 把它 ROP 成 system('/bin/sh') 就解決了