### source code 
```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int choice; // eax
  int boss_hp; // [esp+0h] [ebp-3Ch] BYREF
  const char *boss_name; // [esp+4h] [ebp-38h]
  char description[48]; // [esp+8h] [ebp-34h] BYREF
  int length; // [esp+38h] [ebp-4h]

  init_proc();
  length = 0;
  memset(description, 0, sizeof(description));
  boss_hp = 2147483647;
  boss_name = "Gin";
  while ( 1 )
  {
    while ( 1 )
    {
      while ( 1 )
      {
        while ( 1 )
        {
          menu();
          choice = read_int();
          if ( choice != 2 )
            break;
          power_up(description);
        }
        if ( choice > 2 )
          break;
        if ( choice != 1 )
          goto LABEL_15;
        create_bullet(description);
      }
      if ( choice == 3 )
        break;
      if ( choice == 4 )
      {
        puts("Don't give up !");
        exit(0);
      }
LABEL_15:
      puts("Invalid choice");
    }
    if ( beat(description, &boss_hp) )
      return 0;
    puts("Give me more power !!");
  }
}

int __cdecl create_bullet(char *description)
{
  size_t description_length; // [esp+0h] [ebp-4h]

  if ( *description )
    return puts("You have been created the Bullet !");
  printf("Give me your description of bullet :");
  read_input(description, 48u);
  description_length = strlen(description);
  printf("Your power is : %u\n", description_length);
  *((_DWORD *)description + 12) = description_length;
  return puts("Good luck !!");
}

int __cdecl power_up(char *description)
{
  char s[48]; // [esp+0h] [ebp-34h] BYREF
  size_t v3; // [esp+30h] [ebp-4h]

  v3 = 0;
  memset(s, 0, sizeof(s));
  if ( !*description )
    return puts("You need create the bullet first !");
  if ( *((_DWORD *)description + 12) > 0x2Fu )
    return puts("You can't power up any more !");
  printf("Give me your another description of bullet :");
  read_input(s, 48 - *((_DWORD *)description + 12));
  strncat(description, s, 48 - *((_DWORD *)description + 12));
  v3 = strlen(s) + *((_DWORD *)description + 12);
  printf("Your new power is : %u\n", v3);
  *((_DWORD *)description + 12) = v3;
  return puts("Enjoy it !");
}

int __cdecl beat(int description, int boss_hp)
{
  if ( *(_BYTE *)description )
  {
    puts(">----------- Werewolf -----------<");
    printf(" + NAME : %s\n", *(const char **)(boss_hp + 4));
    printf(" + HP : %d\n", *(_DWORD *)boss_hp);
    puts(">--------------------------------<");
    puts("Try to beat it .....");
    usleep(0xF4240u);
    *(_DWORD *)boss_hp -= *(_DWORD *)(description + 48);
    if ( *(int *)boss_hp <= 0 )
    {
      puts("Oh ! You win !!");
      return 1;
    }
    else
    {
      puts("Sorry ... It still alive !!");
      return 0;
    }
  }
  else
  {
    puts("You need create the bullet first !");
    return 0;
  }
}
```

就是一個打王小遊戲

### **沒有 canary 有開 NX 沒有 FULL RELRO 沒有 FMT exploit**

1. 可以看到武器攻擊力取決於武器的 description 的長度
2. 問題出在 strncat, 這個函數接起來後會在字串最後面放 '\x00'
3. 在 GDB 可以看到 description 後面緊接的就是武器攻擊力, 如下
   
    <img width="986" height="262" alt="image" src="https://github.com/user-attachments/assets/0318d7cb-2e71-4617-8f54-e54283d94e45" />

5. 在 strncat 指定的 size 也是藉由武器攻擊力 (description長度) 決定
6. 為何會覆寫此值？, strncat 會先將他變為 \x00, 然後再加 power_up 寫入的 description 的長度, 所以如果是 47+1 的 case 下, 攻擊力會變成 1
7. 如此一來因為長度在記憶體裡面被覆寫成1, 呼叫 power_up 就能再繼續寫下去, 前 3 byte 寫為 \xff\xff\xff, 這樣攻擊力就會遠大於 boss_hp
8. 然後繼續寫下去直到 return address, leak 出 libc_base 然後跳回 main 再執行一次呼叫 system('/bin/sh') 就好 (one_gadget 也可以)
