### source code 
```c=
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  _QWORD *mmap_addr; // rcx
  __int64 temp; // rdx
  _BYTE buf_2[64]; // [rsp+0h] [rbp-60h] BYREF
  _QWORD buf[2]; // [rsp+40h] [rbp-20h] BYREF
  _BYTE choice[16]; // [rsp+50h] [rbp-10h] BYREF

  init_0();
  fd_202018 = open("/dev/urandom", 0);
  read(fd_202018, buf, 0x10uLL);
  mmap_addr = qword_202020;
  temp = buf[1];
  *qword_202020 = buf[0];
  mmap_addr[1] = temp;
  close(fd_202018);
  while ( 1 )
  {
    write(1, ">> ", 3uLL);
    _read_chk(0LL, choice, 16LL, 16LL);
    if ( choice[0] == '2' )
      break;
    if ( choice[0] == '3' )
    {
      if ( unk_202014 )
        magic_copy(buf_2);
      else
LABEL_13:
        puts("Invalid choice");
    }
    else
    {
      if ( choice[0] != '1' )
        goto LABEL_13;
      if ( unk_202014 )
        unk_202014 = 0;
      else
        login(buf);
    }
  }
  if ( !unk_202014 )
    exit(0);
  memcmp(buf, qword_202020, 0x10uLL);
  return 0LL;
}

int __fastcall login(const char *buf)
{
  size_t length; // rax
  char s[128]; // [rsp+10h] [rbp-80h] BYREF

  printf("Your passowrd :");
  read_input(s, 127u);
  length = strlen(s);
  if ( strncmp(s, buf, length) )
    return puts("Failed !");
  unk_202014 = 1;
  return puts("Login Success !");
}

int __fastcall magic_copy(char *buf)
{
  char src[128]; // [rsp+10h] [rbp-80h] BYREF

  printf("Copy :");
  read_input(src, 63u);
  strcpy(buf, src);
  return puts("It is magic copy !");
}

char *__fastcall read_input(char *a1, unsigned int size)
{
  char *result; // rax
  int v3; // [rsp+1Ch] [rbp-4h]

  v3 = read(0, a1, size);
  if ( v3 <= 0 )
  {
    puts("read error");
    exit(1);
  }
  result = (char *)(unsigned __int8)a1[v3 - 1];
  if ( (_BYTE)result == '\n' )
  {
    result = &a1[v3 - 1];
    *result = 0;
  }
  return result;
}
```
輸入 password 可以使用 magic_copy 的程式

### **有 canary 有開 NX 有 PIE 有 FULL RELRO**
1. 跟之前某一題一樣，login 跟 magic_copy 的 stack 是在同一個區域
2. 可以透過 login 輸入的 password 去控制 magic_copy 裡的 stack, magic_copy 又透過 strcpy 控制到 main 的 stack
3. main 最後的 memcmp 是一種 canary 機制, 若修改到的話會 abort

### solution
1. 我們可以知道 password 輸入失敗並不會退出程式
2. login 裡的 strncmp 的 size 是由我們輸入的 s 的長度而定
3. 基於第 1 點跟第 2 點, 我們可以做到 brute-force, 把 password 暴力破解出來
4. 輸入 16 個 1 把 choice buf 填滿可以 bypass password (我猜是 off-by-one, 等同於將 password 輸入 '\x00', 導致 strlen(password) = 0, 所以 strcmp 自然就過了)
5. 觀察 login 中的 stack 是否有可洩漏的 libc function 在 main 的 password buf 後面, 使用 magic_copy 把它 copy 到 main 上的話 libc_address 一樣可以用 brute-force 把它暴力破解出來 (蓋過 password 沒差, 因為已經有先將 password 暴力破解出來, 到時候再蓋回去就好了)
6. strcpy 碰到 '\x00' 才會停, 所以要把那個 function 前面全部填滿
7. leak 完後一樣先在 password 構造 payload, 再用 magic_copy 讓他 copy 到 main 的 return address
8. 觀察 rdi 為輸入 choice 的那個 buf, 所以輸入 2;sh; 就能開啟 shell

### 原本的 solution
1. 爆破 password 後順便爆破 main 的 saved ebp, 用來計算 pie_base
2. 利用 pie_base 得到 puts_plt 跟 puts_got, ROP 將 puts leak 出來
3. 哪裡不可行? 輸入 password 的 buf 太短, 沒辦法接這麼多的 gadget QQ