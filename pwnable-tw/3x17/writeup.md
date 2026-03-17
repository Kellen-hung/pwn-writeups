### source code 

#### start
IDA 反組譯得知 symbol table 被拔掉... 什麼函數都不知道在哪, 都只看的到 address (gdb 不能直接 b main 超級麻煩)

首要目標一定是先找出 main, 在 IDA 搜尋 mov     rdi, offset, 有這條指令的 function 就是 start function
<img width="588" height="255" alt="image" src="https://github.com/user-attachments/assets/e512d5dc-5885-4b3f-8276-032cad50bc3f" />

然後 r8 那個是 __libc_csu_init, rcx 是 __libc_csu_init, 因為後面用的到所以順便寫出來

#### main
<img width="565" height="312" alt="image" src="https://github.com/user-attachments/assets/39f653d0-2c7d-4d1c-890d-a9b070520750" />


改完變數函數名字後長這樣, 看起來就是個任意寫入

### **有 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**

需要想辦法讓 main function 執行不只一次, 竄改 plt 不可行, 因為根本沒用到其他 function ...

survey 了一下, 如果要讓 main function 反覆執行, 得先追溯到整個 program 的執行流程
#### .init_array 跟 .fini_array
這兩個 array 裡面放的是 function pointer, 屬於 data
#### .init 跟 .fini 跟 __libc_csu.init 跟 __libc_csu.fini
這四個屬於 function, 分別在 main 前後執行
#### 執行流程
.init -> __libc_csu_init -> main -> libc_csu_fini -> .fini
抑或是
.init -> init_array[0] -> init_array [1] ... -> main -> ... fini_array[1] -> fini_array[0] -> .fini

#### 結論
若將 fini_array[1] 改成 main, fini_array[0] 改成 libc_csu_fini, 則程式會無限循環下去, 並且那個 global variable 會一直 +1, 直到他 overflow 後又變成 = 1 可以任意寫入

#### 構建 ROP chain
##### 有一個大問題, ASLR, 無法知道 return address 的位置
解決方法: stack migration, 我一開始將 fini_array[1] 改成 leave; ret, 但這樣 rsp 會跳到不可寫的 segment, 所以不可行, 因此我改成將 fini_array[0] 改成 leave; ret, 這樣 rsp 會在可寫 segment, 得把這個 address 記起來以便構造 ROP chain

##### ROP chain
這個 ELF 用 static linked, 所以很好構造

### solution

先把 main 重複執行, 再 stack migration, 在新的 rsp 位置上建構 ROP chain 進行 syscall execve("/bin/sh", 0, 0)
