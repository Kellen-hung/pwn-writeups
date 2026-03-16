### source code 
```c=
直接看 realloc writeup
```

### **有 canary 有開 NX 有 PIE 有 FULL RELRO**
1. 程式沒有 show 功能, 且 GOT 無法修改，所以要竄改 _IO_2_1_stdout_ 來進行 leak
2. 要有 libc address 可以 leak, 所以要構造出一個 freed unsorted bin chunk
3. 利用 one_gadget 

### solution
主要目標: 把 tcache->fd 改成 stdout, 因為 libc 2.29 會 check p->fd->bk = p, 所以若 malloc unsorted bin 會 SIGSEGV
1. 利用 realloc 特性, 使 chunk overlap
    ```py=
    alloc(0, 0x68, b'A')
    realloc(0, 0)
    realloc(0, 0x18, b'A')
    free(0)
    ```
2. 竄改 size 使他能進入 unsorted bin
3. libc 2.29 會 check p->next 的 prev_inuse bit, 所以要 fake
4. 修改 _IO_2_1_stdout_
5. tcache double free 修改 __realloc_hook
6. 使用 one_gadget