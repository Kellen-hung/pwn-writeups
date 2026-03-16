### source code 
```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int size2; // eax
  unsigned int *nums_ptr; // edi
  char *i; // esi
  unsigned int j; // esi
  int result; // eax
  unsigned int size; // [esp+18h] [ebp-74h] BYREF
  unsigned int nums[8]; // [esp+1Ch] [ebp-70h] BYREF
  _BYTE name[64]; // [esp+3Ch] [ebp-50h] BYREF
  unsigned int canary; // [esp+7Ch] [ebp-10h]

  canary = __readgsdword(0x14u);
  init();
  __printf_chk(1, "What your name :");
  read(0, name, 0x40u);
  __printf_chk(1, "Hello %s,How many numbers do you what to sort :");
  __isoc99_scanf("%u", &size);
  size2 = size;
  if ( size )
  {
    nums_ptr = nums;
    for ( i = 0; (unsigned int)i < size; ++i )
    {
      ((void (__cdecl *)(int, const char *, void *))__printf_chk)(1, "Enter the %d number : ", i);
      fflush(stdout);
      __isoc99_scanf("%u", nums_ptr);
      size2 = size;
      ++nums_ptr;
    }
  }
  sort(nums, size2);
  puts("Result :");
  if ( size )
  {
    for ( j = 0; j < size; ++j )
      ((void (__cdecl *)(int, const char *, void *))__printf_chk)(1, "%u ", (void *)nums[j]);
  }
  result = 0;
  if ( __readgsdword(0x14u) != canary )
    sub_BA0();
  return result;
}

unsigned int __cdecl sort(unsigned int *nums_ptr, int size)
{
  int v2; // ecx
  unsigned int *i; // edi
  unsigned int num1; // edx
  unsigned int num2; // esi
  unsigned int *nums_ptr2; // eax
  unsigned int result; // eax
  unsigned int canary; // [esp+1Ch] [ebp-20h]

  canary = __readgsdword(0x14u);
  puts("Processing......");
  sleep(1u);
  if ( size != 1 )
  {
    v2 = size - 2;
    for ( i = &nums_ptr[size - 1]; ; --i )
    {
      if ( v2 != -1 )
      {
        nums_ptr2 = nums_ptr;
        do
        {
          num1 = *nums_ptr2;
          num2 = nums_ptr2[1];
          if ( *nums_ptr2 > num2 )
          {
            *nums_ptr2 = num2;
            nums_ptr2[1] = num1;
          }
          ++nums_ptr2;
        }
        while ( i != nums_ptr2 );
        if ( !v2 )
          break;
      }
      --v2;
    }
  }
  result = __readgsdword(0x14u) ^ canary;
  if ( result )
    sub_BA0();
  return result;
}
```

就是排序

### **有 canary 有開 NX 有 FULL RELRO 沒有 FMT exploit**

1. 測試時發現輸入 name 後 printf 出來會噴亂碼
-> 沒有將原本記憶體清空
2. 看 source code 時發現輸入要 sort 幾個數字時會 overflow

### solution

1. 先找 name 記憶體裡面有用的資訊進行 leak
![image](https://hackmd.io/_uploads/BJh_Pjzpyx.png)
![image](https://hackmd.io/_uploads/ByacDjM61g.png)
0xf7f82000 是 .got.plt 的初始 address, 利用
```
readelf -S ./libc_32.so.6 | grep '.got.plt'
```
可以找到 offset, 利用 leak 出來的值 - offset = libc_base, 之後再計算 system 跟 /bin/sh 的 address
(這個 leak 前面要填多少 payload 本地跟遠端不一樣, 慢慢試)

2. 觀察過後發現 sort function 是傳 pointer 過去進行 sort, 所以會直接動到在 main 時的 stack
3. 想覆蓋 return address 勢必得 leak 出 canary 或者不動到 canary, 在這題中 leak 不出來, 只能想如何保留 canary
4. scanf("%u",&num); 很特別, 他除了輸入數字不會 error 外, 還可以輸入 + 跟 -, 因為要讓你能輸入 -100 +234 之類的, 輸入其他字母會直接噴錯, 而且如果是輸入 + 或 - 的話原本 memory 的值不會變動
5. 依上所述, 我們只要找到需要幾個比 canary 小的數字, 需要幾個比 system 小且比 canary 大的數字, 需要幾個比 binsh 小且比 system 大的數字, 就可以讓 sort 幫我們排好在 main stack 裡面的結構, ret 時呼叫 system("/bin/sh")