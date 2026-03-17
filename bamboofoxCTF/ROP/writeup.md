## syscall 需要 register 代表不同意思
[x86_32 syscall table](https://gist.github.com/GabriOliv/a9411fa771a1e5d94105cb05cbaebd21)

找到 execve

<img width="836" height="61" alt="image" src="https://github.com/user-attachments/assets/10030fb5-b1f8-4145-bac1-4f274e34d0c2" />

分別是 eax = 0x0b, ebx = &name, ecx = edx = 0
