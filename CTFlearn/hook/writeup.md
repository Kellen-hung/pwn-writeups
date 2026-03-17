### source code 
```c=
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <malloc.h>
#include <unistd.h>
#include <stdint.h>

#define MAX_CHUNKS 6

uint64_t read_num(){
    char input[32];
    memset(input, 0, sizeof(input));
    read(STDIN_FILENO, input, 0x1F);
    return strtoull(input, 0, 10);
}

void menu() {
    unsigned int idx = 0;
    uint64_t index;
    char *chunks[MAX_CHUNKS];
    memset(chunks, 0, MAX_CHUNKS * sizeof(char *));

    while(1){
        printf("\n1) malloc %u/%u\n", idx, MAX_CHUNKS);
        puts("2) free");
        puts("3) edit");
        puts("4) quit");

        printf("> ");
        long c = read_num();

        switch (c)
        {
        case 1:
            if(idx >= MAX_CHUNKS){
                puts("Maximum requests reached");
                return;
            }

            printf("Size: ");
            uint64_t size = read_num();
            chunks[idx] = malloc(size);
            if(chunks[idx]){
                printf("Data: ");
                read(STDIN_FILENO, chunks[idx++], size);
            }else{
                puts("malloc failed");
            }
            break;
        
        case 2:
            printf("Index: ");
            index = read_num();
            if(index >= MAX_CHUNKS){
                puts("Invalid chunk");
                break;
            }

            free(chunks[index]);
            break;
        
        case 3:
            printf("Index: ");
            index = read_num();
            if(index >= MAX_CHUNKS){
                puts("Invalid chunk");
                break;
            }

            if(chunks[index]){
                printf("Data: ");
                read(STDIN_FILENO, chunks[index], size);
            }else{
                puts("Empty index");
            }

            break;

        case 4:
            return;
        
        default:
            puts("Invalid option");
            break;
        }
    }
}

int main(void){
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);

    printf("puts @ %p\n", puts);

    menu();

	return 0;
}

```

### **有開 canary 有開 NX 有 FULL RELRO 沒有 FMT exploit**

看 source code 可以發現有很嚴重的 UAF 問題

**tcache attack** - tcache 很像 fastbin, 就是個單純的 single linked-list 結構, 他在 get (malloc) 的時候不會去 check size bit, 所以比 fastbin attack 簡單, 建議參考[源碼](https://elixir.bootlin.com/glibc/glibc-2.31/source/malloc/malloc.c#L2934)
**Use after free** - 修改 fd, 破壞 single linked-list 的結構, 
**__free_hook** - call free 時若 __free_hook 不是 NULL, 則 rip 會跳轉到 __free_hook 指向的指令

### solution

1. 先 malloc 兩個 chunk, 為什麼要兩個？因為到時候要把 __free_hook 的位置從 tcache 取出來, 如果只 malloc 一個, 後面要 malloc 第二次的時候系統會認為 tcache 已經沒東西, 導致她直接再從 heap 切一個 chunk 給你
![image](https://hackmd.io/_uploads/r12Gc9Wh1x.png)
圖中第一個 chunk 為 tache base (source code 裡名字為 key), 第四個是 top chunk
2. 把這兩個 chunk free 掉, 這樣 tcache count = 2
![image](https://hackmd.io/_uploads/SkOPc5b2yx.png)
3. 修改 chunk[1] 的 fd 為 __free_hook address (一定要 [1], 不然 __free_hook 又會跑到 tcache 的尾巴導致 malloc 取不出來)
![image](https://hackmd.io/_uploads/By7Us5W3kx.png)
可以看到第三個 chunk 的 fd 值是 __free_hook 的 address 了
![image](https://hackmd.io/_uploads/S1sis9Wn1l.png)
tcache bin 的第二個 chunk 變成 __free_hook
4. 再來 malloc 兩次把 __free_hook malloc 出來並把他寫入 system 的 address (tcache bin 是 LIFO)
![image](https://hackmd.io/_uploads/S19FacZ3ye.png)
5. 再 malloc 一塊 chunk 讓它的 data 是 /bin/sh 字串, 然後 call free 就成功執行 shell 了!!!

p.s. 我一開始只 malloc 一個, 然後一直沒辦法把 __free_hook 從 tcache 拿出來, 中風