## syscall 需要 register 代表不同意思
[x86_32 syscall table](https://gist.github.com/GabriOliv/a9411fa771a1e5d94105cb05cbaebd21)

找到 execve
![image](https://hackmd.io/_uploads/H1ldGDe4yl.png)
分別是 eax = 0x0b, ebx = &name, ecx = edx = 0