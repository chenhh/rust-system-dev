# 記憶體洩漏

## 簡介

在C++中，如果引用計數智慧指針出現了迴圈引用，就會導致記憶體洩漏。

而Rust中也一樣存在引用計數智慧指針Rc，那麼Rust中是否可能製造出記憶體洩漏呢？ Yes：可能出現記憶體洩漏，但是要特別操作，一般情況下不易發生。

## 記憶體洩漏屬於記憶體安全

在程式設計語言設計這個層面，**“**<mark style="color:red;">**記憶體洩漏”是一個基本上無法在編譯階段徹底解決的問題（通常在執行期才會明顯出現）**</mark>。在許多場景下，什麼是“記憶體洩漏”、什麼不是“記憶體洩漏”，本身就沒有一個完全客觀的評判標準。它實質上跟程式師的“意圖”有關。程式很難自動判定出哪些變數是以後還會繼續用的，哪些是不再被使用的。

在C++和Rust中是一樣的，<mark style="color:red;">**如果出現了迴圈引用，那麼只能通過手動打破迴圈的方式來解決記憶體洩漏的問題**</mark>。編譯器無法通過靜態檢查來保證你不會犯這個錯誤。

記憶體洩漏顯然是一種bug。但它跟“記憶體不安全”這種bug的性質不一樣。“記憶體洩漏”是對“正常資料”的“應該執行但是沒有執行”的操作，“記憶體不安全”是對“不正常資料”的“不應該執行但是執行了”的操作。從後果上說，“記憶體不安全”導致的後果比“記憶體洩漏”要嚴重得多。

|     | 正常資料  | 不正常資料  |
| --- | ----- | ------ |
| 執行了 | 正常    | 記憶體不正常 |
| 沒執行 | 記憶體洩漏 | 正常     |

程式語言的設計者當然是希望能徹底解決記憶體洩漏的問題。但是很可惜，這個問題恐怕不是在語言層面能徹底解決的問題。**所謂“徹底解決”的意思是，用戶無論使用何種技巧，永遠無法構造出記憶體洩漏的情況。Rust語言無法給出這樣的保證**。