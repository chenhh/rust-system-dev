# python asyncio

## Python AsyncIO

Python是3.4以後，在標準上逐步加入了asyncio、async與await等支援。

AsyncIO 的概念最早起源於 JavaScript 的 async/await 語法。它在單執行緒中實現了類似於多執行緒的效果，採用類似協程的方式在背景執行。由事件循環（event loop）在有空閒時檢查各個背景任務是否有返回結果，適合處理涉及網路傳輸的 I/O bound 任務。

例如 API 伺服器。Server首先接受多個使用者的請求，然後等到處理完後自動回傳給使用者。若API Server不支援 AsyncIO的話，可能會導致對使用者來說的請求處理時間過長，遲遲沒有等不到回應。

由於受限Python 全域鎖GIL(Global Interpreter Lock) 的特性，當我們打算在Python運行多執行緒的程式碼時所有的執行緒都會被鎖住，導致沒辦法執行，效果相當於單執行緒。

## yield的歷史

探討asyncio的文件中經常看到對協程的探討，基本上都是從yield的介紹開始，而yield最常的作用之一，就是作為生成器（Generator）。需要時才產生一個值，這是產生器（Generator）的概念。

實際上，一個函式若有yield，傳回的產生器就實現了迭代器協定（然而迭代器不一定就是產生器），不過，這時還稱不上是個協程，只不過是能在yield時暫停流程，並將指定值及流程交給呼叫方。

在PEP 255提出之後，yield的作用是作為產生器，而Python中也有許多的產生器實作，目的通常是實現惰性求值，以在某些場合提升效率。

#### green thread

在使用執行緒時，雖然可以透過等待、通知之類的機制，來進行某種程度的流程控制，不過，很大部份還是由作業系統決定了切換的時機。

<mark style="color:red;">而在使用Python的yield、next、send等之時，流程的切換是由開發者決定，相較於執行緒來說，協程的成本很低，而且是在一個執行緒之中完成</mark>，因此有時會看到以微執行緒（microthread）、綠色執行緒（green thread）等方式，來形容某些類型的協程實現。

運用協程時，如果有方式能夠在被阻斷時，就將流程還給呼叫方，在某個事件發生時，再推動產生器繼續後面之流程，那麼也就能夠使用協程來實現並行。

因此除了使用yield建立產生器之外，這時還需要一個主迴圈，不斷地檢查並處理事件，在某些事件發生時推進產生器，而原本會阻斷的程式庫，必須替換為可非同步執行的實作，這可以透過select之類的模組來實現。

asyncio要明確地使用@coroutine、yield from，而後來Python 3.5改用async與await，至於一個可以await的協程，就是awaitable物件。

## 使用asyncio

asyncio本身主要有兩個對象：直接使用（end-user）的開發者與框架設計者。龐大的API文件中，大部份都是給框架設計者看的，直接使用的開發者只要了解async與await關鍵字的使用時機即可。

```python
# -*- coding: UTF-8 -*-
# python 3.11
import asyncio
import time


def block_dosomething(i):
    print(f"第 {i} 次開始")
    time.sleep(1)
    print(f"第 {i} 次結束")


def block_main():
    start = time.time()
    for i in range(5):
        block_dosomething(i + 1)
    # 5 secs
    print(f"block time: {(time.time() - start):.2f} (s)")


async def dosomething(i):
    print(f"第 {i} 次開始")
    await asyncio.sleep(1)
    print(f"第 {i} 次結束")


def async_main():
    start = time.time()
    # 包裝成tasks, 不可直接傳coroutine
    tasks = (asyncio.create_task(dosomething(i + 1)) for i in range(5))
    asyncio.run(asyncio.wait(tasks))
    # 1 sec
    print(f"async time: {(time.time() - start):.2f} (s)")


if __name__ == "__main__":
    block_main()
    async_main()
```







## 參考資料

* [https://www.ithome.com.tw/voice/107416](https://www.ithome.com.tw/voice/107416)
* [https://www.ithome.com.tw/voice/138875](https://www.ithome.com.tw/voice/138875)
