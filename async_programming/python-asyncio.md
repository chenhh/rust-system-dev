# python asyncio

## Python AsyncIO

Python是3.4以後，在標準上逐步加入了asyncio、async與await等支援。為了簡化並更好地標識非同步IO，從Python 3.5開始引入了新的語法async和await，可以讓coroutine的程式碼更簡潔易讀。

<mark style="color:red;">asyncio模組內部實現了EventLoop，把需要執行的協程扔到EventLoop中執行，就實現了非同步IO</mark>。

AsyncIO 的概念最早起源於 JavaScript 的 async/await 語法。它在單執行緒中實現了類似於多執行緒的效果，採用類似協程的方式在背景執行。由事件循環（event loop）在有空閒時檢查各個背景任務是否有返回結果，適合處理涉及網路傳輸的 I/O bound 任務。

例如 API 伺服器。Server首先接受多個使用者的請求，然後等到處理完後自動回傳給使用者。若API Server不支援 AsyncIO的話，可能會導致對使用者來說的請求處理時間過長，遲遲沒有等不到回應。

由於受限Python 全域鎖GIL(Global Interpreter Lock) 的特性，當我們打算在Python運行多執行緒的程式碼時所有的執行緒都會被鎖住，導致沒辦法執行，效果相當於單執行緒。

<figure><img src="../.gitbook/assets/image (20).png" alt="" width="451"><figcaption><p>python asyncio</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image.png" alt="" width="405"><figcaption><p>非同步程式在Requests發出後，所有權交回給主行程/執行緒，等待Response完成後再通知主行程</p></figcaption></figure>

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

## 協程 (Coroutine)&#x20;

協程是一個比較特殊的函式， 也是為瞭解決單執行緒應用之中的等待浪費資源問題， 與常規函式不同，協程可以在適當的時機暫停、恢復和互動執行，這種能力使得協程特別適合處理非同步的任務，比如 I/O 操作、網路請求等需要等待的工作。

async def 明確告知 Python 該函式/方法具有非同步執行的能力，即定義為協程。

另外 await 語法則是被用來告知 Python 可以在此處暫停執行 coroutine 轉而執行其他工作，而<mark style="color:red;">且該語法只能在 coroutine 函式內使用，因此 async def 與 await 通常會一起出現</mark>。

### 使用asyncio

asyncio本身主要有兩個對象：直接使用（end-user）的開發者與框架設計者。龐大的API文件中，大部份都是給框架設計者看的，<mark style="background-color:red;">直接使用的開發者只要了解async與await關鍵字的使用時機即可</mark>。

async與await 這兩個語法糖主要是讓我們更直觀的標示非同步的函式與執行的進入點。

* <mark style="background-color:red;">async：用來宣告函式能夠有非同步的功能</mark>。 <mark style="color:red;">以async定義的函數為協程</mark>。凡是用async def 定義的 都要用await去呼叫。不可以直接使用。如果直接呼叫，只會返回一個協程物件。
* <mark style="background-color:red;">await：用來標記非同步的執行，只能存在於協程內</mark>。將所有權交出(還給原行程/執行緒?)，不會阻塞，可執行其它(在事件迴圈中)的非同步程式，等待(事件迴圈)執行結果回傳後再繼續執行下去。
* 另 1 個關於 await 語法的重點是 <mark style="color:red;">await 之後只能接 awaitables 物件</mark>，例如 coroutine 或者是之後會介紹到的 Task, Future 以及有實作 **await**() 方法 的物件，所以不是所有的物件或操作都能夠用 await 進行暫停。

```python
# -*- coding: UTF-8 -*-
# async01.py
import asyncio


# 函數前宣告async即為coroutine
async def myfunc():
    await asyncio.sleep(1)
    print('hello')


if __name__ == '__main__':
    # myfunc() #RuntimeWarning: coroutine 'myfunc' was never awaited
    # coroutine必須用run執行，無法直接執行函數
    asyncio.run(myfunc())
```

## 事件迴圈(event loop)

事件迴圈是 asyncio 模組的核心(可視為任務的調度員)，用以負責執行非同步(asynchronous)的工作。事件迴圈實例提供了註冊、取消和執行任務和回呼的方法。

把一些非同步函數 (就是任務，Task，一會就會說到) 註冊到這個事件循環上，事件迴圈會循環執行這些函數 (但同時只能執行一個)，當執行到某個函數時，如果它正在等待 I/O 返回(或是其它await函數)，事件迴圈會暫停它的執行去執行其他的函數；當某個函數完成 I/O 後會恢復，下次循環到它的時候繼續執行。因此，這些非同步函數可以協同 (Cooperative) 運行：這就是事件迴圈的目標。

而實際上事件迴圈是 Python 類別 BaseEventLoop ，正如其名， 事件迴圈關鍵運作部分是 1 個無窮迴圈（原始碼），不斷地 loop 進行排程/執行非同步任務、回調函式(callbacks)等工作， I/O 類的工作十分適合以非同步方式交由事件迴圈執行，例如網路通訊、檔案讀寫等等，以利 event loop 進行工作切換。

原則上，我們不需針對事件迴圈進行太多操作與干涉。可以使用 [asyncio.get\_running\_loop()](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get\_running\_loop) 或者 [asyncio.get\_event\_loop()](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get\_event\_loop) 即可取得 event loop 實例(instance)以進行操作。

### 事件迴圈流程

非同步程式要處理的是IO-bound的問題。

同步程式中，如果要取得A、B、C、D四個資料，就要A->B->C->D按照順序執行。

如果是非同步的程式， A–> B–> C–> D 基本上就是類似先丟出需求之後再去等回應。

簡單的說事件迴圈就是：

1. 建立事件迴圈。
2. 在事件迴圈上註冊任務(Tasks)。

<mark style="color:red;">既然非同步程式可以在多個任務之間切換，一定有個列表包含所有的任務，而這個任務列表和機制就稱為事件迴圈</mark>。

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="466"><figcaption><p>事件迴圈任務列表輪詢</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (2).png" alt="" width="469"><figcaption><p>事件迴圈任務完成後執行回調函數(必須是非阻塞函數)</p></figcaption></figure>

## Awaitables物件

awaitables 關鍵字就代表著以下 3 種 Python 物件(objects)，也是 await 語法適用的對象：

* 協程(coroutines)
* 任務(asks) - asyncio.Task&#x20;
* Futures - asyncio.Future

asyncio 很多函式/方法(method)所需要的參數多半是上述 3 種不同類型的物件之一，因此一定要注意其差別，如果是 3 種皆可，通常會在文件中以 aw , \*aws 或者 awaitables 說明。

### 任務(tasks)

在事件迴圈中，工作的執行是以任務為單位， 事件迴圈一次僅會執行一個任務，如果某個 任務正在等待執行結果，也就是執行到 await 的地方，那麼事件迴圈將會暫停(suspend)並將之進行排程，接著切換執行其他任務，回調函數(callback)或者執行某些 I/O 相關的操作。

我們可以將任務視為是協程的再包裝，因此可以看到`asyncio.create_task(coroutine)` 函數接受的參數必須是協程。

任務是非同步執行的單位，負責作為事件迴圈和協程物件的溝通介面，經過任務物件的包裝才能被事件迴圈執行。

<mark style="background-color:red;">直接 await task 不會對並行有幫助</mark>。

```python
# -*- coding: UTF-8 -*-
import asyncio


async def coro():
    print('hello')
    await asyncio.sleep(1)
    print('world')


if __name__ == '__main__':
    # 取得事件迴圈後再run
    # loop = asyncio.get_event_loop()
    # task = loop.create_task(coro())
    # loop.run_until_complete(task)

    # 也可直接用run
    asyncio.run(coro())
```

```python
# -*- coding: UTF-8 -*-

import asyncio
import time


async def say_after(delay: int, what: str) -> None:
    await asyncio.sleep(delay)
    print(what)


async def main_coroutine() -> None:
    await say_after(1, 'hello')
    await say_after(2, 'world')
    # coroutine沒經過task包裝不會節省時間, 3秒


async def main_create_task() -> None:
    task1 = asyncio.create_task(
        say_after(1, 'hello'))
    task2 = asyncio.create_task(
        say_after(2, 'world'))
    await task1
    await task2
    # 經過task包裝可節省時間, 2秒

async def main_gather() -> None:
    c1 = say_after(1, 'hello')
    c2 = say_after(2, 'world')
    await asyncio.gather(c1, c2)
    # 使用gather可節省時間, 2秒


def time_elapsed(func: callable) -> None:
    start = time.perf_counter()
    asyncio.run(func())
    print(f"used: {time.perf_counter() - start:.2f} s")


if __name__ == '__main__':
    time_elapsed(main_coroutine)
    time_elapsed(main_create_task)
    time_elapsed(main_gather)
```

### 任務可以取消

從結果可以發現執行 task.cancel() 之前， 任務就已經開始執行了，這是由於 await asyncio.sleep(5) 給了 事件迴圈切換執行 cancel\_me() 的機會，所以我們才會看到在 cancel\_me(): sleep 出現在 main(): call cancel 之前。

如果把 await asyncio.sleep(5) 註解掉就會看到 cancel\_me() 連執行的機會都沒有就被取消了。

換言之，在呼叫 asyncio.create\_task() 後， 事件迴圈就已經接收到有 1 個 Task 需要執行，只是該 協程需要有機會被事件迴圈切換執行，而剛好 await asyncio.sleep(5) 恰好給了事件迴圈切換執行的機會。

```python
# -*- coding: UTF-8 -*-
import asyncio


async def cancel_me():
    print('cancel_me(): sleep')
    try:
        # Wait for 1 hour
        await asyncio.sleep(3600)
    except asyncio.CancelledError:
        print('cancel_me(): cancel sleep')
        raise
    finally:
        print('cancel_me(): after sleep')


async def main():
    print('main(): running')
    # Create a "cancel_me" Task
    task = asyncio.create_task(cancel_me())

    # Wait for 5 second
    print('main(): sleep')
    await asyncio.sleep(5)

    print('main(): call cancel')
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print('main(): cancel_me is cancelled now')


if __name__ == '__main__':
    asyncio.run(main())

    # main(): running
    # main(): sleep
    # cancel_me(): sleep
    # main(): call
    # cancel
    # cancel_me(): cancel
    # sleep
    # cancel_me(): after
    # sleep
    # main(): cancel_me is cancelled
    # now
```

### Futures

Task 繼承自 Future，因此 Future 是相對底層(low-level)的 awaitable Python 物件，用以代表非同步操作的最終結果，一般並不需要自己創造 Future 物件進行操作，多以 coroutine 與 Task (界面比較容易使用)為主。

不過仍有些 asyncio 模組的函式會回傳 Future 物件 ，例如 asyncio.run\_coroutine\_threadsafe() 回傳的就是 Future 物件。

與 Task 不同的是， Future 物件並不是對 coroutine 進行再包裝，而是作為代表非同步操作最終結果的物件，因此該物件有 1 個 set\_result() 方法，可以將結果寫入，同時該 Future 物件也會被標為結束(done)的狀態，所以 Future 物件通常會與 coroutines 或 Tasks 混搭使用。

， Future 也與 Task 相同擁有取消 Task, 新增/刪除回呼函數等方法可供使用。

```python
import asyncio


async def do_async_job(fut):
    await asyncio.sleep(2)
    fut.set_result('Hello future')


async def main():
    loop = asyncio.get_running_loop()

    future = loop.create_future()
    loop.create_task(do_async_job(future))

    # Wait until future has a result
    await future

    print(future.result())


if __name__ == '__main__':
    asyncio.run(main())
```

## asyncio.gather()

asyncio.gather 用來並行運行任務。

asyncio.gather() 方法，其傳入參數為 \*aws 即代表 awaitable 物件，所以我們可以同時傳入 coroutine, Task 甚至 Future 物件皆可， asyncio.gather() 會自動處理 awaitables, 例如將 coroutine 統一轉為 Task。

asyncio.gather() 作用在於執行多個 awaitable 物件（一樣是透過 event loop），並收集每 1 個的回傳值存於一個串列(list)中。因此實務上可以將多個執行相同任務的 coroutines, Tasks 或者 Futures 一起交由 asyncio.gather() 執行。

```python
import asyncio
import threading
from datetime import datetime
import random

async def do_async_job() -> int:
    await asyncio.sleep(2)
    print(datetime.now().isoformat(), 'thread id', threading.current_thread().ident)
    return random.randint(1, 10)

async def main():
    # 依順序進行，每一次都會sleep，沒有切換工作
    # await do_async_job()
    # await do_async_job()
    # await do_async_job()

    # 使用gather使task可非同步切換
    job1 = do_async_job()
    job2 = do_async_job()
    job3 = do_async_job()
    return_values = await asyncio.gather(job1, job2, job3)
    for v in return_values:
        print(f'result => {v}')


asyncio.run(main())
```

## 設定時限(Timeouts)

對於事件迴圈來說，能夠持續切換執行不同工作的能力是相當重要的，如果有任何一個工作佔據事件迴圈使其無法進行切換，那麼就會拖延其他工作的執行，導致其他工作都要等佔據事件迴圈的工作完成才能繼續。

可以使用 asyncio.wait\_for() 為 awaitables(coroutine, Task, Future) 設定 1 個時限。

```python
import asyncio


async def do_async_job():
    await asyncio.sleep(2)
    print('never print')


async def main():
    try:
        await asyncio.wait_for(do_async_job(), timeout=1)
    except asyncio.TimeoutError:
        print('timeout!')


if __name__ == '__main__':
    asyncio.run(main())
    # timeout!
```

我們可以利用 asyncio.wait\_for() 為某個工作設定時限，當該工作超出時限時，捕捉 TimeoutError 執行備選方案。

```python
import asyncio


async def do_async_job():
    await asyncio.sleep(2)
    print("do async job 1")

async def do_async_job_2nd_plan():
    await asyncio.sleep(1)
    print("do async job 2")


async def main():
    try:
        await asyncio.wait_for(do_async_job(), timeout=1)
    except asyncio.TimeoutError:
        await asyncio.wait_for(do_async_job_2nd_plan(), timeout=2)


if __name__ == '__main__':
    asyncio.run(main())
    # do async job 2

```

## asyncio.to\_thread()

由於事件迴圈負責執行非同步的工作，為了發揮事件迴圈最大效能，我們都需要確保每一個協程中都需要有 await 的存在，或者想辦法將執行時間很長的部分轉為協程，使得事件迴圈能夠有機會切換執行其他工作的機會，否則事件迴圈遇到執行特別時間長的程式碼，又沒有 await 能夠讓事件迴圈能夠轉為執行其他工作時，就會造成事件迴圈阻塞。

```python
import asyncio
import threading

from time import sleep


def hard_work():
    print('thread id:', threading.get_ident())
    sleep(3)


async def do_async_job():
    hard_work()
    await asyncio.sleep(1)
    print('job done!')


async def main():
    task1 = asyncio.create_task(do_async_job())
    task2 = asyncio.create_task(do_async_job())
    task3 = asyncio.create_task(do_async_job())
    await asyncio.gather(task1, task2, task3)


if __name__ == '__main__':
    # ask1, task2, task3 其實都各花 3 秒在 sleep ，
    # 同時也阻塞其他工作的進行，因此造成這個範例花費約 9 秒
    asyncio.run(main())

```



## asyncio.run()的行為

asyncio.run 是 Python 3.7 新加的語法。等同於以下的寫法：

```python
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

簡單來說，asyncio.run會取得事件迴圈、透過run\_until\_complete執行指定的協程，這會阻斷直到指定的（主）協程完成（也就是執行完該協程函式定義的流程），之後會收集尚未沒完成的任務（主協程中可能又建立了其他任務，然而沒有await這些任務），取消這些任務（會在各協程函式中引發CancelledError），然後，再次使用run\_until\_complete執行這些任務（等待CancelledError善後處理完成），最後關閉迴圈。

而且，想取得事件迴圈中全部的任務，我們可以透過asyncio.all\_tasks，想從這群任務中取得尚未沒完成的任務，可以透過asyncio.gather。認識這兩個函式，對於控制事件迴圈來說，是很重要的，而且，要注意的是，它們收集的是一組Task，並不會包含Future。

```python
# -*- coding: UTF-8 -*-
# python 3.11
import asyncio
import time


def block_dosomething(i):
    print(f"第 {i} 次開始")
    time.sleep(1)    // 會在此處阻塞
    print(f"第 {i} 次結束")


def block_main():
    start = time.time()
    for i in range(5):
        block_dosomething(i + 1)
    # 5 secs，因為每次呼叫函數都阻塞1秒
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
    # 1 sec, 因為每次呼叫函數時，只要sleep就釋放所有權處理下一個函數，因此等待時間重疊。
    print(f"async time: {(time.time() - start):.2f} (s)")


if __name__ == "__main__":
    block_main()
    async_main()
```







## 參考資料

* [https://www.ithome.com.tw/voice/107416](https://www.ithome.com.tw/voice/107416)
* [https://www.ithome.com.tw/voice/138875](https://www.ithome.com.tw/voice/138875)
* [https://docs.python.org/zh-tw/3/library/asyncio.html](https://docs.python.org/zh-tw/3/library/asyncio.html)
* [https://myapollo.com.tw/blog/asyncio-how-event-loop-works/](https://myapollo.com.tw/blog/asyncio-how-event-loop-works/)
* [https://myapollo.com.tw/blog/begin-to-asyncio/](https://myapollo.com.tw/blog/begin-to-asyncio/)
* [https://www.dongwm.com/post/understand-asyncio-1/#asyncio%E5%B9%B6%E5%8F%91%E7%9A%84%E6%AD%A3%E7%A1%AE/%E9%94%99%E8%AF%AF%E5%A7%BF%E5%8A%BF](https://www.dongwm.com/post/understand-asyncio-1/#asyncio%E5%B9%B6%E5%8F%91%E7%9A%84%E6%AD%A3%E7%A1%AE/%E9%94%99%E8%AF%AF%E5%A7%BF%E5%8A%BF)
