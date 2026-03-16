### source code 
```c=
void __fastcall __noreturn main(char *a1, char **a2, char **a3)
{
  char choice[8]; // [rsp+0h] [rbp-28h] BYREF
  unsigned __int64 canary; // [rsp+8h] [rbp-20h]

  canary = __readfsqword(0x28u);
  init_0();
  while ( 1 )
  {
    menu();
    read(0, choice, 4uLL);
    switch ( (unsigned int)strtol(choice, 0LL, 10) )
    {
      case 1u:
        raise_flower();
        break;
      case 2u:
        visit_garden();
        break;
      case 3u:
        remove_flower();
        break;
      case 4u:
        clean_garden();
        break;
      case 5u:
        puts("See you next time.");
        exit(0);
      default:
        puts("Invalid choice");
        break;
    }
  }
}

int raise_flower()
{
  struct flowers *flower; // rbx
  char **name; // rbp
  flowers **next; // rcx
  int index; // edx
  struct infos info; // [rsp+4h] [rbp-24h] BYREF

  *(_QWORD *)&info.canary = __readfsqword(0x28u);
  info.name_size = 0;

  if ( flower_num > 99u )
    return puts("The garden is overflow");

  flower = (struct flowers *)malloc(0x28uLL);
  flower->is_existed = 0LL;
  flower->name = 0LL;
  *(_QWORD *)flower->color = 0LL;
  *(_QWORD *)&flower->color[8] = 0LL;
  *(_QWORD *)&flower->color[16] = 0LL;

  __printf_chk(1LL, "Length of the name :");
  if ( (unsigned int)__isoc99_scanf("%u", &info) == -1 )
    exit(-1);

  name = (char *)malloc(info.name_size);
  if ( !name )
  {
    puts("Alloca error !!");
    exit(-1);
  }

  __printf_chk(1LL, "The name of flower :");
  read(0, name, info.name_size);
  flower->name = name;

  __printf_chk(1LL, "The color of the flower :");
  __isoc99_scanf("%23s", flower->color);

  LODWORD(flower->is_existed) = 1;
  if ( garden[0] )
  {
    next = &garden[1];
    index = 1;
    while ( *next )
    {
      ++index;
      ++next;
      if ( index == 100 )
        goto LABEL_13;
    }
  }
  else
  {
    index = 0;
  }
  garden[index] = flower;
LABEL_13:
  ++flower_num;
  return puts("Successful !");
}

void __fastcall visit_garden()
{
  __int64 index; // rbx
  flowers *node; // rax

  index = 0LL;
  if ( flower_num )
  {
    do
    {
      node = garden[index];
      if ( node )
      {
        if ( LODWORD(node->is_existed) )
        {
          __printf_chk(1LL, "Name of the flower[%u] :%s\n", index, node->name);
          __printf_chk(1LL, "Color of the flower[%u] :%s\n", index, garden[index]->color);
        }
      }
      ++index;
    }
    while ( index != 100 );
  }
  else
  {
    puts("No flower in the garden !");
  }
}

int remove_flower()
{
  flowers *node; // rax
  unsigned int index; // [rsp+4h] [rbp-14h] BYREF
  unsigned __int64 canary; // [rsp+8h] [rbp-10h]

  canary = __readfsqword(0x28u);
  if ( !flower_num )
    return puts("No flower in the garden");
  __printf_chk(1LL, "Which flower do you want to remove from the garden:");
  __isoc99_scanf("%d", &index);
  if ( index <= 0x63 && (node = garden[index]) != 0LL )
  {
    LODWORD(node->is_existed) = 0;
    free(garden[index]->name);
    return puts("Successful");
  }
  else
  {
    puts("Invalid choice");
    return 0;
  }
}

unsigned __int64 clean_garden()
{
  flowers **flower_addr; // rbx
  flowers *flower; // rdi
  unsigned __int64 canary; // [rsp+8h] [rbp-20h]

  canary = __readfsqword(0x28u);
  flower_addr = garden;
  do
  {
    flower = *flower_addr;
    if ( *flower_addr && !LODWORD(flower->is_existed) )
    {
      free(flower);
      *flower_addr = 0LL;
      --flower_num;
    }
    ++flower_addr;
  }
  while ( flower_addr != &garden[100] );
  puts("Done!");
  return __readfsqword(0x28u) ^ canary;
}
```
輸入 password 可以使用 magic_copy 的程式

### **有 canary 有開 NX 有 PIE 有 FULL RELRO**
1. ```c=
    struct flowers
    {
      unsigned __int64 is_existed;
      char **name;
      char color[24];
    };

    struct infos
    {
      unsigned int name_size;
      unsigned int canary;
      unsigned int canary1;
      unsigned int data[6];
    };

    ```
2. remove_flower 中的 free(garden[index]->name); 有 UAF 漏洞

### solution
1. 標準的 heap 題, 因 size 是自訂的, 所以可以用 unsorted bin leak 出 libc
2. 比較特別的是這題在 malloc(size) 之前會先 malloc(0x28)
3. unsorted bin 理論上是 FIFO, 但我用 GDB 發現他是從 size 小的開始 split (我不知道為什麼)
4. 總之就是搭配 gdb 讓 malloc(size) 的時候剛好可以從 unsorted bin 分配出來, 並且不要改掉 bk, 用 visit leak 出 bk
5. 有了 libc 之後, 就用 fastbin double free 任意寫入 __malloc_hook
6. fastbin malloc 的時候會檢查 size, 可以用 libc hardcode 特性 (首個 byte 一定是 0x7f) 當作 size, 因為它不檢查 alignment, 且 64 位元下 0x7f 等同 size = 0x70, fastbin 也不會因為首四個 bit 為 1 而 crash
7. 搭配 one_gadget get shell, 但會發現每個 gadget 的條件都不符合, 但 rsp+0x50 那個 gadget 如果觸發 double free error, 可以 get shell, 因為 libc 處理 double free 錯誤的時候會呼叫 malloc, 而那時條件會剛好符合