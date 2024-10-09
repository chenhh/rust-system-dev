# 非同步程式

##

## python的非同步函數（協程）

在3.5過後，我們可以使用async修飾將普通函數和生成器函數包裝成非同步函數和非同步生成器。

```python
# coroutine
async def async_function():
    return 1
    
# async generator
async def async_generator():
    yield 1

# 通過類型判斷可以驗證函數的類型
import types
print(type(function) is types.FunctionType)
print(type(generator()) is types.GeneratorType)
print(type(async_function()) is types.CoroutineType)
print(type(async_generator()) is types.AsyncGeneratorType)

# 直接呼叫非同步函數不會返回結果，而是返回一個協程物件
print(async_function())
# <coroutine object async_function at 0x102ff67d8>

# 協程需要通過其他方式來驅動，因此可以使用這個協程物件的send方法給協程傳送一個值
print(async_function().send(None))

# 但是會丟出異常。
# 因為生成器/協程在正常返回退出時會拋出一個StopIteration異常，
# 而原來的返回值會存放在StopIteration物件的value屬性中，
# 通過以下捕獲可以獲取協程真正的返回值
try:
    async_function().send(None)
except StopIteration as e:
    print(e.value)
# 1
```

通過上面的方式來新建一個run函數來驅動協程函數。

```python
def run(coroutine):
    try:
        coroutine.send(None)
    except StopIteration as e:
        return e.value
```

## await語法

在協程函數中，可以通過await語法來掛起自身的協程，並等待另一個協程完成直到返回結果：

```python
async def async_function():
    return 1

async def await_coroutine():
    result = await async_function()
    print(result)
    
run(await_coroutine())
# 1
```

要注意的是，await語法只能出現在通過async修飾的函數中，否則會報SyntaxError錯誤。

而且await後面的物件需要是一個Awaitable，或者實現了相關的協議。

檢視Awaitable抽象類的程式碼，表明了只要一個類實現了\_\_await\_\_方法，那麼通過它構造出來的實例就是一個Awaitable。

而且可以看到，Coroutine類也繼承了Awaitable，而且實現了send，throw和close方法。所以await一個呼叫非同步函數返回的協程對像是合法的。



