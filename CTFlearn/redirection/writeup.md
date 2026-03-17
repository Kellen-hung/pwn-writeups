### source code 
```c=
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void ask_user(){
    char buf[128];
    memset(buf, 0, sizeof(buf));
    system("echo Input your message: ");
    fgets(buf, sizeof(buf), stdin);
    printf(buf);
    puts("Thanks for sending the message!");
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);

    ask_user();

    return 0;
}
```

### **有開啟 canary 有開 NX**

雖然可以利用 format string exploit 找出 canary 的值, 但找完程式就結束了, 沒用

### solution

1. puts裡的字不是/bin/sh的話, 程式只跑一次鐵定不夠的, 因此先將 puts@got 裡改成 main, 並可以順手將 printf@got 改成 system, 因為buf 是可控的
2. 跑第二圈的時候直接送 /bin/sh 就會變 system("/bin/sh")了