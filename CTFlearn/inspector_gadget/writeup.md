### source code 
```c=
int __fastcall main(int argc, const char **argv, const char **envp)
{
  char buf[32]; // [rsp+10h] [rbp-20h] BYREF

  setvbuf(_bss_start, 0LL, 2, 0LL);
  setvbuf(stdin, 0LL, 2, 0LL);
  write(1, "You have one job!\n", 0x13uLL);
  read(0, buf, 0x100uLL);
  return 0;
}
```

### **沒開啟 canary 有開 NX**

ROP

### solution

1. ROPgadget 出來的 gadget 完全不夠, 查資料說可以用 __libc_csu_init 裡的 gadget
![image](https://hackmd.io/_uploads/ByIKSEhqyg.png)
2. 如果有 pop r12; pop r13; pop r14; ret 的話就可以控制 rdx, rdi, rsi了, 利用 write(1,*write_got,6) 以及 write(1,*read_got,6) 印出 write 以及 read 的真實地址, 丟進 libc database 找 libc 版本, 然後算 system 跟 /bin/sh 的位置, 最後 return 回 main 再 ROP 一次就可以拿到 shell
3. padding 要用幾byte 請用 GDB 慢慢看 