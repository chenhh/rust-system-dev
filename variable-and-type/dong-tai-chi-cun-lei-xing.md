# 動態尺寸類型

大多數的類型都有一個在編譯時就已知的固定尺寸，並實現了 `trait Sized`。<mark style="color:blue;">只有在執行時才知道尺寸的類型稱為動態尺寸類型(dynamically sized type)（DST）</mark>，或者非正式地稱為非固定尺寸類型(unsized type)。

切片和 trait物件是 DSTs 的兩個例子：

* 指向 DST 的指針類型的尺寸是固定的(sized)，但是<mark style="color:red;">是指向固定尺寸類型的指針的尺寸的</mark><mark style="color:red;background-color:red;">兩倍</mark>。&#x20;
  * 指向切片的指針也存儲了切片的元素的數量。&#x20;
  * 指向 trait物件的指針也存儲了一個指向虛函數表(vtable)的指針。
* 當接受了 ?Sized約束時，DST 可以作為類型實參( type arguments)使用。默認情況下，任何類型形參(type parameter)都擁有 Sized約束。
* 可以為 DST 實現 trait。與類型參數中的默認設置不同，在 trait定義中默認存在 Self: ?Sized約束。
* 結構體可以包含一個 DST 作為最後一個欄位，這使得該結構體也成為是一個 DST。

注意：變量、函數參數、常量項和靜態項必須是 Sized。

