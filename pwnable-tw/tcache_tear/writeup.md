### source code 
```c=
void __fastcall __noreturn main(__int64 argc, char **argv, char **envp)
{
  __int64 choice; // rax
  unsigned int free_count; // [rsp+Ch] [rbp-4h]

  init_0(argc, argv, envp);
  printf("Name:");
  read_name(&name, 32LL);
  free_count = 0;
  while ( 1 )
  {
    while ( 1 )
    {
      menu();
      choice = read_long();
      if ( choice != 2 )
        break;
      if ( free_count <= 7 )
      {
        free(ptr);
        ++free_count;
      }
    }
    if ( choice > 2 )
    {
      if ( choice == 3 )
      {
        info();
      }
      else
      {
        if ( choice == 4 )
          exit(0);
LABEL_14:
        puts("Invalid choice");
      }
    }
    else
    {
      if ( choice != 1 )
        goto LABEL_14;
      allocate();
    }
  }
}

int allocate()
{
  size_t v0; // rax
  int size; // [rsp+8h] [rbp-8h]

  printf("Size:");
  v0 = read_long();
  size = v0;
  if ( v0 <= 0xFF )
  {
    ptr = malloc(v0);
    printf("Data:");
    read_name((__int64)ptr, size - 16);
    LODWORD(v0) = puts("Done !");
  }
  return v0;
}

ssize_t info()
{
  printf("Name :");
  return write(1, &name, 32uLL);
}
```

heap (tcache)? 題

### **有 canary 有開 NX 有 FULL RELRO 沒有 FMT exploit**
1. 很明顯 free 後有 UAF 問題
2. allocate size, 但輸入 size 卻是 size - 16 ?
3. 測試如果將 size 輸入為 < 16, malloc 會取一塊 size 為 0x20 的 chunk, 但卻能寫超過 16 個字 (存在 heap overflow)
4. 不能寫 GOT，只能 leak libc_base 了
5. unsorted_bin 如果只有一個 chunk, fd 跟 bk 會指到 main_arena+0x60
6. main_arena 在 libc 裡面, 所以可以用 unsorted_bin leak libc

### solution

1. 先 double free
2. 在 name 的位置做一個會進 unsorted_bin 的 fake chunk
3. 這個 fake chunk 的 prev_inuse bit 要為 1, 代表 prev 使用中
4. 這個 fake chunk 的 next 也要是使用中，也就是 fake chunk -> next -> next 的 prev_inuse bit 要為 1
5. 如果沒有做 3 跟 4, free 這個 fake chunk 的時候會試著合併 freed chunk, 呼叫 unlink 導致 error
6. 呼叫 info 洩漏 libc 位址
7. 再 double free 一次修改 free_hook
8. 再 malloc 一個 chunk, data = /bin/sh, free 掉它就會執行 system('/bin/sh')