### source code 
```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  signal(14, timeout);
  alarm(0x3Cu);
  memset(&myCart, 0, 0x10u);
  menu();
  return handler();
}

unsigned int handler()
{
  char nptr[22]; // [esp+16h] [ebp-22h] BYREF
  unsigned int v2; // [esp+2Ch] [ebp-Ch]

  v2 = __readgsdword(0x14u);
  while ( 1 )
  {
    printf("> ");
    fflush(stdout);
    my_read(nptr, 21);
    switch ( atoi(nptr) )
    {
      case 1:
        list();
        break;
      case 2:
        add();
        break;
      case 3:
        delete();
        break;
      case 4:
        cart();
        break;
      case 5:
        checkout();
        break;
      case 6:
        puts("Thank You for Your Purchase!");
        return __readgsdword(0x14u) ^ v2;
      default:
        puts("It's not a choice! Idiot.");
        break;
    }
  }
}

int list()
{
  puts("=== Device List ===");
  printf("%d: iPhone 6 - $%d\n", 1, 199);
  printf("%d: iPhone 6 Plus - $%d\n", 2, 299);
  printf("%d: iPad Air 2 - $%d\n", 3, 499);
  printf("%d: iPad Mini 3 - $%d\n", 4, 399);
  return printf("%d: iPod Touch - $%d\n", 5, 199);
}

unsigned int add()
{
  Cart *node; // [esp+1Ch] [ebp-2Ch]
  char nptr[22]; // [esp+26h] [ebp-22h] BYREF
  unsigned int v3; // [esp+3Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  printf("Device Number> ");
  fflush(stdout);
  my_read(nptr, 21);
  switch ( atoi(nptr) )
  {
    case 1:
      node = create("iPhone 6", 199);
      insert(node);
      goto LABEL_8;
    case 2:
      node = create("iPhone 6 Plus", 299);
      insert(node);
      goto LABEL_8;
    case 3:
      node = create("iPad Air 2", 499);
      insert(node);
      goto LABEL_8;
    case 4:
      node = create("iPad Mini 3", 399);
      insert(node);
      goto LABEL_8;
    case 5:
      node = create("iPod Touch", 199);
      insert(node);
LABEL_8:
      printf("You've put *%s* in your shopping cart.\n", node->name);
      puts("Brilliant! That's an amazing idea.");
      break;
    default:
      puts("Stop doing that. Idiot!");
      break;
  }
  return __readgsdword(0x14u) ^ v3;
}

unsigned int delete()
{
  int v1; // [esp+10h] [ebp-38h]
  int MyCart_8; // [esp+14h] [ebp-34h]
  int number; // [esp+18h] [ebp-30h]
  Cart *next; // [esp+1Ch] [ebp-2Ch]
  struct Cart *prev; // [esp+20h] [ebp-28h]
  char nptr[22]; // [esp+26h] [ebp-22h] BYREF
  unsigned int canary; // [esp+3Ch] [ebp-Ch]

  canary = __readgsdword(0x14u);
  v1 = 1;
  MyCart_8 = Cart_8;
  printf("Item Number> ");
  fflush(stdout);
  my_read(nptr, 21);
  number = atoi(nptr);
  while ( MyCart_8 )
  {
    if ( v1 == number )
    {
      next = *(Cart **)(MyCart_8 + 8);
      prev = *(struct Cart **)(MyCart_8 + 12);
      if ( prev )
        prev->next = next;
      if ( next )
        next->prev = prev;
      printf("Remove %d:%s from your shopping cart.\n", v1, *(const char **)MyCart_8);
      return __readgsdword(0x14u) ^ canary;
    }
    ++v1;
    MyCart_8 = *(_DWORD *)(MyCart_8 + 8);
  }
  return __readgsdword(0x14u) ^ canary;
}

int cart()
{
  int count; // eax
  int v2; // [esp+18h] [ebp-30h]
  int total_price; // [esp+1Ch] [ebp-2Ch]
  Cart *i; // [esp+20h] [ebp-28h]
  _BYTE v5[22]; // [esp+26h] [ebp-22h] BYREF
  unsigned int canary; // [esp+3Ch] [ebp-Ch]

  canary = __readgsdword(0x14u);
  v2 = 1;
  total_price = 0;
  printf("Let me check your cart. ok? (y/n) > ");
  fflush(stdout);
  my_read(v5, 21u);
  if ( v5[0] == 'y' )
  {
    puts("==== Cart ====");
    for ( i = (Cart *)Cart_8; i; i = i->next )
    {
      count = v2++;
      printf("%d: %s - $%d\n", count, i->name, i->price);
      total_price += i->price;
    }
  }
  return total_price;
}

unsigned int checkout()
{
  int total_price; // [esp+10h] [ebp-28h]
  Cart v2; // [esp+18h] [ebp-20h] BYREF
  unsigned int canary; // [esp+2Ch] [ebp-Ch]

  canary = __readgsdword(0x14u);
  total_price = cart();
  if ( total_price == 7174 )
  {
    puts("*: iPhone 8 - $1");
    asprintf(&v2.name, "%s", "iPhone 8");
    v2.price = 1;
    insert(&v2);
    total_price = 7175;
  }
  printf("Total: $%d\n", total_price);
  puts("Want to checkout? Maybe next time!");
  return __readgsdword(0x14u) ^ canary;
}
```

就是一個購物程式

### **有 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**
1. cart 的結構長這樣
    ```c=
    struct Cart
    {
      char *name; // asprintf (malloc)
      int price;
      struct Cart *next;
      struct Cart *prev;
    };
    ```
2. 在 checkout 可以看到如果消費總額剛好為 7174 元, 會送你一台 1$ 的 iPhone 8, 但這個 iPhone 8 的 node 沒有 malloc, 而是在 stack 上的 local variable 並串接在 myCart 的 double-linked list 上
3. 第一點情況會導致一旦 stack 資料被覆蓋的話程式會壞掉, 如果另一個 function 剛好可以在位置進行輸入, 則因為有 printf(node->name), 造成任意讀取, 這隻程式就有這個問題, 示意圖如下
    <img width="602" height="973" alt="image" src="https://github.com/user-attachments/assets/07d204cd-bfe0-429c-8f2b-42fc941c761b" />
    可以看到在 delete function 下, input 可以完美覆蓋iPhone 8 的所有資料

### solution

1. 首先先湊到 7174 元, ~~(我直接請 chatGPT 幫我找)~~
2. 再來就是要 leak 出 libc_base, 只要將 iPhone8 的 name 蓋成 puts_got, next 跟 prev 蓋成 0 就好
3. 有 libc 之後要想怎麼利用 delete 那邊酷似 unlink 的機制進行攻擊
4. 這邊看起來只能覆寫 atoi 的 GOT
5. 在 delete function 要 leave ret 的時候, 將 ebp 遷到 atoi_got + 0x22 的地方
    (這意味著要將 saved ebp 修改成 atoi_got + 0x22),
    因為利用 GDB 知道當 handler 呼叫 my_read 的時候, buf 的位置是在 ebp - 0x22
6. 如何拿到 delete function 的 ebp ?
7. 利用 libc 裡面一個叫做 environ 的指標, 此指標指向 stack 上的環境變數
8. 利用此指標的值減掉 offset 即可得到 delete function 的 ebp
9. unlink 攻擊, 將 next 設為 delete_ebp - 12, prev 設為 atoi_got + 0x22
10. leave; ret; 後, ebp 就被改為 atoi_got + 0x22 了, 在 handler 下次呼叫 my_read 時候可以任意寫 atoi 的 GOT, 將它寫成 system address
11. GDB 發現 call system 時參數為 system address 本身, 因此只要再後面多加 || /bin/sh\x00 就可以順利開啟 shell
