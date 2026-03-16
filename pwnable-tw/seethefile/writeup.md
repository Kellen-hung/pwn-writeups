### source code 
```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int result; // eax
  char nptr[32]; // [esp+Ch] [ebp-2Ch] BYREF
  unsigned int canary; // [esp+2Ch] [ebp-Ch]

  canary = __readgsdword(0x14u);
  init();
  welcome();
  while ( 1 )
  {
    menu();
    __isoc99_scanf("%s", nptr);
    switch ( atoi(nptr) )
    {
      case 1:
        openfile();
        break;
      case 2:
        readfile();
        break;
      case 3:
        writefile();
        break;
      case 4:
        closefile();
        break;
      case 5:
        printf("Leave your name :");
        __isoc99_scanf("%s", name);
        printf("Thank you %s ,see you next time\n", name);
        if ( fp )
          fclose(fp);
        exit(0);
        return result;
      default:
        puts("Invaild choice");
        exit(0);
        return result;
    }
  }
}

int openfile()
{
  if ( fp )
  {
    puts("You need to close the file first");
    return 0;
  }
  else
  {
    memset(magicbuf, 0, 400u);
    printf("What do you want to see :");
    __isoc99_scanf("%63s", filename);
    if ( strstr(filename, "flag") )
    {
      puts("Danger !");
      exit(0);
    }
    fp = fopen(filename, "r");
    if ( fp )
      return puts("Open Successful");
    else
      return puts("Open failed");
  }
}

size_t readfile()
{
  size_t result; // eax

  memset(magicbuf, 0, 400u);
  if ( !fp )
    return puts("You need to open a file first");
  result = fread(magicbuf, 0x18Fu, 1u, fp);
  if ( result )
    return puts("Read Successful");
  return result;
}

int writefile()
{
  if ( strstr(filename, "flag") || strstr(magicbuf, "FLAG") || strchr(magicbuf, 125) )
  {
    puts("you can't see it");
    exit(1);
  }
  return puts(magicbuf);
}

int closefile()
{
  int result; // eax

  if ( fp )
    result = fclose(fp);
  else
    result = puts("Nothing need to close");
  fp = 0;
  return result;
}
```

[FILE 介紹](https://ctf-wiki.org/pwn/linux/user-mode/io-file/introduction/)

### **沒有 canary 有開 NX 沒有 FULL RELRO**
1. case 5 的 scanf('%s',name) 明顯可以 overflow
2. overflow name 可以覆蓋 fp
3. name 是 global variable, 而且 main 也不會 return, 所以就算沒有 canary, stack overflow 也不會發生

### solution

first one
1. 既然可以覆寫 fp, 那意味著可以自行構造一個 fake file stream
2. 將 vtable 構造成 system
3. 呼叫 fclose() 的時候觸發 system('/bin/sh') ('/bin/sh' 用 GDB 看放在哪)
4. 如何 leak libc base? 讀寫 /proc/self/maps 

second one
1. 既然可以覆寫 fp, 那意味著可以自行構造一個 fake file stream
2. 將 vtable 的地址構造成可控區域, 並將那區域的 function pointer 全部蓋成 system
3. 注意 fake file 裡的 flag & 0x2000 != 0 時, 才會進入 vtable -> close
4. 呼叫 fclose() 的時候觸發 vtable -> system('/bin/sh') (本來為 close)