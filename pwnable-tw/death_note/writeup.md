### source code 
```c=
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  int choice; // eax

  setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 2, 0);
  while ( 1 )
  {
    while ( 1 )
    {
      menu();
      choice = read_int();
      if ( choice != 2 )
        break;
      show_note();
    }
    if ( choice > 2 )
    {
      if ( choice == 3 )
      {
        del_note();
      }
      else
      {
        if ( choice == 4 )
          exit(0);
LABEL_13:
        puts("Invalid choice");
      }
    }
    else
    {
      if ( choice != 1 )
        goto LABEL_13;
      add_note();
    }
  }
}

unsigned int add_note()
{
  int index; // [esp+8h] [ebp-60h]
  char name[80]; // [esp+Ch] [ebp-5Ch] BYREF
  unsigned int canary; // [esp+5Ch] [ebp-Ch]

  canary = __readgsdword(0x14u);
  printf("Index :");
  index = read_int();
  if ( index > 10 )
  {
    puts("Out of bound !!");
    exit(0);
  }
  printf("Name :");
  read_input(name, 80u);
  if ( !is_printable(name) )
  {
    puts("It must be a printable name !");
    exit(-1);
  }
  note[index] = strdup(name);
  puts("Done !");
  return __readgsdword(0x14u) ^ canary;
}

int __cdecl is_printable(char *s)
{
  size_t i; // [esp+Ch] [ebp-Ch]

  for ( i = 0; strlen(s) > i; ++i )
  {
    if ( s[i] <= '\x1F' || s[i] == '\x7F' )
      return 0;
  }
  return 1;
}

void __cdecl show_note()
{
  int index; // [esp+Ch] [ebp-Ch]

  printf("Index :");
  index = read_int();
  if ( index > 10 )
  {
    puts("Out of bound !!");
    exit(0);
  }
  if ( note[index] )
    printf("Name : %s\n", note[index]);
}

void __cdecl del_note()
{
  int index; // [esp+Ch] [ebp-Ch]

  printf("Index :");
  index = read_int();
  if ( index > 10 )
  {
    puts("Out of bound !!");
    exit(0);
  }
  free(note[index]);
  note[index] = 0;
}
```

### **有 canary 沒開 NX 沒 FULL RELRO**
1. add 的 i 有漏洞, i > 8 表示 i 可以等於 8, 但實際上 book 應該是 book[0] ~ book[7]
2. book 的下一個變數是 sizes, 所以可以透過 book 的 oob 覆蓋 sizes[0] 的值, 使其變成一個 heap address

### solution

1. 就是在 puts@got 的位置構造 shellcode
    [Hacking/Shellcode/Alphanumeric/x86 printable opcodes](https://web.archive.org/web/20110716082815/http://skypher.com/wiki/index.php?title=X86_alphanumeric_opcodes#16-_or_32-bit_register.2Fmemory_values)
    [Linux Shellcode - Alphanumeric Execve()](https://blackcloud.me/Linux-shellcode-alphanumeric/)
    [Pwnable.tw Death Note Writeup](https://kazma.tw/2024/06/07/Pwnable-tw-Death-Note-Writeup/)