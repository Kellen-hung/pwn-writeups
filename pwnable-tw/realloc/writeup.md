### source code 
```c=
int __fastcall __noreturn main(int argc, const char **argv, const char **envp)
{
  int choice; // [rsp+4h] [rbp-Ch] BYREF
  unsigned __int64 canary; // [rsp+8h] [rbp-8h]

  canary = __readfsqword(0x28u);
  choice = 0;
  init_proc(argc, argv, envp);
  while ( 1 )
  {
    while ( 1 )
    {
      menu();
      __isoc99_scanf("%d", &choice);
      if ( choice != 2 )
        break;
      reallocate();
    }
    if ( choice > 2 )
    {
      if ( choice == 3 )
      {
        rfree();
      }
      else
      {
        if ( choice == 4 )
          _exit(0);
LABEL_13:
        puts("Invalid Choice");
      }
    }
    else
    {
      if ( choice != 1 )
        goto LABEL_13;
      allocate();
    }
  }
}

int allocate()
{
  _BYTE *temp; // rax
  unsigned __int64 index; // [rsp+0h] [rbp-20h]
  unsigned __int64 size; // [rsp+8h] [rbp-18h]
  void *chunk_addr; // [rsp+18h] [rbp-8h]

  printf("Index:");
  index = read_long();
  if ( index > 1 || heap[index] )
  {
    LODWORD(temp) = puts("Invalid !");
  }
  else
  {
    printf("Size:");
    size = read_long();
    if ( size <= 0x78 )
    {
      chunk_addr = realloc(0LL, size);
      if ( chunk_addr )
      {
        heap[index] = chunk_addr;
        printf("Data:");
        temp = (_BYTE *)(heap[index] + read_input(heap[index], (unsigned int)size));
        *temp = 0;
      }
      else
      {
        LODWORD(temp) = puts("alloc error");
      }
    }
    else
    {
      LODWORD(temp) = puts("Too large!");
    }
  }
  return (int)temp;
}

int reallocate()
{
  unsigned __int64 index; // [rsp+8h] [rbp-18h]
  unsigned __int64 size; // [rsp+10h] [rbp-10h]
  void *chunk; // [rsp+18h] [rbp-8h]

  printf("Index:");
  index = read_long();
  if ( index > 1 || !heap[index] )
    return puts("Invalid !");
  printf("Size:");
  size = read_long();
  if ( size > 0x78 )
    return puts("Too large!");
  chunk = realloc((void *)heap[index], size);
  if ( !chunk )
    return puts("alloc error");
  heap[index] = chunk;
  printf("Data:");
  return read_input(heap[index], size);
}

int rfree()
{
  _QWORD *v0; // rax
  unsigned __int64 index; // [rsp+8h] [rbp-8h]

  printf("Index:");
  index = read_long();
  if ( index > 1 )
  {
    LODWORD(v0) = puts("Invalid !");
  }
  else
  {
    realloc((void *)heap[index], 0LL);
    v0 = heap;
    heap[index] = 0LL;
  }
  return (int)v0;
}
```

就是一個玩 realloc 的程式

### **有 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**
1. realloc : 
    ```c=
    chunk_addr = realloc(ptr, size);
    // 如果 ptr = NULL, 等同於 malloc(size)
    // 如果 size = 0, 等同於 free(ptr)
    // 如果兩者都不為 0, 將 ptr 的大小改為 size
    ```
2. 在 rfree 裡面沒有 UAF, 但注意：reallocate 裡如果 size 指定為 0, 則會有 UAF 問題出現
3. libc 版本為 2.29, 因此 tcache 有防 double free
    ```c=
    #if USE_TCACHE
      {
        size_t tc_idx = csize2tidx (size);
        if (tcache != NULL && tc_idx < mp_.tcache_bins)
          {
        /* Check to see if it's already in the tcache.  */
        tcache_entry *e = (tcache_entry *) chunk2mem (p);

        /* This test succeeds on double free.  However, we don't 100%
           trust it (it also matches random payload data at a 1 in
           2^<size_t> chance), so verify it's not an unlikely
           coincidence before aborting.  */
        if (__glibc_unlikely (e->key == tcache))
          {
            tcache_entry *tmp;
            LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
            for (tmp = tcache->entries[tc_idx];
             tmp;
             tmp = tmp->next)
              if (tmp == e)
            malloc_printerr ("free(): double free detected in tcache 2");
            /* If we get here, it was a coincidence.  We've wasted a
               few cycles, but don't abort.  */
          }

        if (tcache->counts[tc_idx] < mp_.tcache_count)
          {
            tcache_put (p, tc_idx);
            return;
          }
          }
      }
    #endif
    ```
    可以看到 e->key == tcache 才會再進入偵測 double free
4. tcache 不檢查 size 欄位是否正確
5. got hijack
6. printf 執行完後 rax 的值為印出的字元數

### solution

1. 做到後面會知道需要改 atoll_got 兩次, 所以需要在 tcache bin 找兩個 idx 一取出來就是 atoll_got 
2. 其中 size = 0x20 是必要的, 因為 read_long 只讓你輸入 22 bytes, 所以 printf 不會讓 rax > 22, 導致 size 無法指定
3. 再來需要將 heap[0] 跟 heap[1] 都清乾淨, 後面才能 allocate
4. 將 atoll_got 改成 printf_plt, 因為 atoll 後面的參數可控
5. 這會導致 atoll(buf) 變成 printf(buf), format string exploit 出 libc_base
6. 輸入 %7\$p 的 stdout 或 %21\$p 的 __libc_start_main 來 leak 都可以
7. 接著就是要 allocate size = 0x20 那一塊 chunk, 將 atoll 改為 system
8. 觸發 atoll('/bin/sh') 就完成了