# release如何去除常數偵錯資訊

{% embed url="https://rustcc.cn/article?id=db0645ea-d3b8-4da5-a2a0-adffa3321b48" %}

沒啥好辦法，實踐上一般在某個容器/chroot環境建構發佈，這樣最後也不會洩漏什麼重要的路徑資訊，你可以看看你包管理器（homebrew/pacman/apt..)安裝的rust程式。

反正就是開發環境建構的release別直接發佈。
