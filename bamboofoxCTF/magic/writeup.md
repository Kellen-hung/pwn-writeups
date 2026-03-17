第14行+6為了跳過這三行 (0d是CR字元)

<img width="644" height="100" alt="image" src="https://github.com/user-attachments/assets/5c388b59-4b1b-48af-8603-2f72f267ba3c" />

架構圖

<img width="509" height="458" alt="image" src="https://github.com/user-attachments/assets/414ede89-4682-4e3e-bd07-c8cf7327eb4e" />

## 知識點
1. strlen()遇到 \x00 會停
2. scanf()遇到回車，換行會停
3. 只要在injection input前多加 \x00 不被strlen()讀進去就不會在do_magic function被洗亂
