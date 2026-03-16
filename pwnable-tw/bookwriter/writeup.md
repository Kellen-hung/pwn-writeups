### source code 
```c=
void __fastcall __noreturn main(int a1, char **a2, char **a3)
{
  setvbuf(stdout, 0LL, 2, 0LL);
  puts("Welcome to the BookWriter !");
  author_input();
  while ( 1 )
  {
    menu();
    switch ( read_long() )
    {
      case 1LL:
        add();
        break;
      case 2LL:
        view();
        break;
      case 3LL:
        edit();
        break;
      case 4LL:
        info();
        break;
      case 5LL:
        exit(0);
      default:
        puts("Invalid choice");
        break;
    }
  }
}

__int64 author_input()
{
  printf("Author :");
  return read_int(author_name, 0x40u);
}

int add()
{
  unsigned int index; // [rsp+Ch] [rbp-14h]
  char *page; // [rsp+10h] [rbp-10h]
  __int64 size; // [rsp+18h] [rbp-8h]

  for ( index = 0; ; ++index )
  {
    if ( index > 8 )
      return puts("You can't add new page anymore!");
    if ( !book[index] )
      break;
  }
  printf("Size of page :");
  size = read_long();
  page = (char *)malloc(size);
  if ( !page )
  {
    puts("Error !");
    exit(0);
  }
  printf("Content :");
  read_int(page, size);
  book[index] = page;
  sizes[index] = size;
  ++count;
  return puts("Done !");
}

__int64 read_long()
{
  char nptr[24]; // [rsp+10h] [rbp-20h] BYREF
  unsigned __int64 canary; // [rsp+28h] [rbp-8h]

  canary = __readfsqword(0x28u);
  if ( (int)read_int(nptr, 0x10u) < 0 )
  {
    puts("read error");
    exit(1);
  }
  return (unsigned int)atol(nptr);
}

__int64 __fastcall read_int(char *buf, unsigned int a2)
{
  int chk; // [rsp+1Ch] [rbp-4h]

  chk = _read_chk(0LL, buf, a2, a2);
  if ( chk < 0 )
  {
    puts("read error");
    exit(1);
  }
  if ( buf[chk - 1] == '\n' )
    buf[chk - 1] = 0;
  return (unsigned int)chk;
}

int view()
{
  unsigned int index; // [rsp+Ch] [rbp-4h]

  printf("Index of page :");
  index = read_long();
  if ( index > 7 )
  {
    puts("out of page:");
    exit(0);
  }
  if ( !book[index] )
    return puts("Not found !");
  printf("Page #%u \n", index);
  return printf("Content :\n%s\n", (const char *)book[index]);
}

int edit()
{
  unsigned int index; // [rsp+Ch] [rbp-4h]

  printf("Index of page :");
  index = read_long();
  if ( index > 7 )
  {
    puts("out of page:");
    exit(0);
  }
  if ( !book[index] )
    return puts("Not found !");
  printf("Content:");
  read_int((__int64)book[index], sizes[index]);
  sizes[index] = strlen((const char *)book[index]);
  return puts("Done !");
}

unsigned __int64 info()
{
  int choice; // [rsp+4h] [rbp-Ch] BYREF
  unsigned __int64 canary; // [rsp+8h] [rbp-8h]

  canary = __readfsqword(0x28u);
  index = 0;
  printf("Author : %s\n", author_name);
  printf("Page : %u\n", count);
  printf("Do you want to change the author ? (yes:1 / no:0) ");
  _isoc99_scanf("%d", &choice);
  if ( choice == 1 )
    author_input();
  return __readfsqword(0x28u) ^ canary;
}
```

### **有 canary 有開 NX 沒 PIE 有 FULL RELRO**
1. add 裡的 index 寫成 > 8, 這代表可以有 book[8], 但 book 只有 8 個 index (0~7)
2. 如果寫到 book[8], 則會 overwrite 掉 sizes[0] 的值 (變為某個 heap address)
3. author name 如果寫滿, 可以 leak 出 heap_base, 因為沒有處理 \x00, 所以導致 printf 會繼續把資料讀出來
4. 如果 top chunk size < malloc size, libc 會把 top chunk 丟進去 unsorted bin, 然後再自己跟 system 要足夠大的 chunk 
5. malloc 一個 chunk, 會遍歷 unsorted_bin, 一個一個比對是不是剛好 size 的 chunk, 若不是的話會丟進 small(large)_bin 做排序
6. 如果找完一圈還沒找到, 就會根據要的 size 進 small(large)_bin 找並進行切割, 切割完剩下的 chunk 又會被丟回 unsorted bin 
7. 查了之後知道完全就是 ***House of Orange***

### solution
[神教學，步驟基本上一樣](https://www.cnblogs.com/xshhc/p/17327672.html)
但是我 local 跑不過, server 才跑得過