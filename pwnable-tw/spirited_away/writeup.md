### source code 
```c=
int survey()
{
  char s[56]; // [esp+10h] [ebp-E8h] BYREF
  size_t nbytes; // [esp+48h] [ebp-B0h]
  size_t length; // [esp+4Ch] [ebp-ACh]
  char comment[80]; // [esp+50h] [ebp-A8h] BYREF
  int age; // [esp+A0h] [ebp-58h] BYREF
  char *name; // [esp+A4h] [ebp-54h]
  char reason[80]; // [esp+A8h] [ebp-50h] BYREF

  nbytes = 60;
  length = 80;
LABEL_2:
  memset(comment, 0, sizeof(comment));
  name = (char *)malloc(60u);

  printf("\nPlease enter your name: ");
  fflush(stdout);
  read(0, name, nbytes);

  printf("Please enter your age: ");
  fflush(stdout);
  __isoc99_scanf("%d", &age);

  printf("Why did you came to see this movie? ");
  fflush(stdout);
  read(0, reason, length);

  fflush(stdout);
  printf("Please enter your comment: ");
  fflush(stdout);
  read(0, comment, nbytes);

  ++cnt;

  printf("Name: %s\n", name);
  printf("Age: %d\n", age);
  printf("Reason: %s\n", reason);
  printf("Comment: %s\n\n", comment);
  fflush(stdout);

  sprintf(s, "%d comment so far. We will review them as soon as we can", cnt);
  puts(s);
  puts(&::s);
  fflush(stdout);

  if ( cnt > 199 )
  {
    puts("200 comments is enough!");
    fflush(stdout);
    exit(0);
  }
  while ( 1 )
  {
    printf("Would you like to leave another comment? <y/n>: ");
    fflush(stdout);
    read(0, &choice, 3u);
    if ( choice == 'Y' || choice == 'y' )
    {
      free(name);
      goto LABEL_2;
    }
    if ( choice == 'N' || choice == 'n' )
      break;
    puts("Wrong choice.");
    fflush(stdout);
  }

  puts("Bye!");
  return fflush(stdout)
}
```

### **沒 canary 有開 NX 沒 FULL RELRO**
1. reason 沒有 memset to 0
2. name 可以 heap overflow (why? 反正沒用)
3. `sprintf(s, "%d comment so far. We will review them as soon as we can", cnt);` 剛好 56 個字, cnt >= 10 會 overflow
4. 如果 cnt 是三位數, nbytes 會變 0x6e

### solution
1. 第一個 cycle 去 leak libc address (因為 reason 沒有 memset)
2. 第二個 cycle 去 leak survey 的 ebp (因為計算 fake chunk 位置覆蓋在 name 上面)
3. 接下來把 cnt 搞成 100, 進行 overflow
4. 在 reason 的位置做 fake chunk (因為 reason 沒有 memset, 資料不會被洗掉)
5. 這樣下一個 cycle 的 name 就會從 fastbin 取到 fake chunk 的 address
6. 再下一個 cycle 輸入 name 的時候就可以 overflow 到 return address, 呼叫 system('/bin/sh')