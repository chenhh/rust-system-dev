# slint

在專案中使用 Slint 需要兩個元件：

1. `.slint` 檔，其中包含以 Slint 語言編寫的使用者介面的文本描述。
2. 嵌入 .slint 檔案的項目的程式設計語言。

## slint檔

Slint 語言描述了使用 Slint 框架的可擴展圖形用戶介面。可以使用 Slint 語言編寫使用者介面，並將其保存在擴展名為 .slint 的檔中。

每個 .slint 檔案定義一個或多個元件(components)。這些元件聲明一個元素樹。元件構成了 Slint 中組合的基礎。使用它們來構建您自己的可重用UI控制項集。您可以將其名稱下每個聲明的元件用作另一個元件中的元素。

```java
component MyButton inherits Text {
    color: black;
    // ...
}

export component MyApp inherits Window {
    preferred-width: 200px;
    preferred-height: 100px;
    Rectangle {
        width: 200px;
        height: 100px;
        background: green;
    }
    hello := MyButton {
        x:0;y:0;
        text: "hello";
    }
    world := MyButton {
        y:0;
        x: 50px;
        text: "world";
    }
}
```

MyButton 和 MyApp 都是元件。Window 和 Rectangle 是 MyApp 使用的內置元素。MyApp 還將 MyButton 元件重新用作兩個單獨的元素。

元素具有屬性，您可以為其分配值。上面的示例將字串常量 「hello」 分配給第一個 MyButton 的 text 屬性。您還可以分配整個表達式。當表達式所依賴的任何屬性發生變化時，Slint 會重新計算表示式，這使得使用者介面是反應性的。

您可以使用 ：= 語法命名元素。

某些元素也可以通過預定義的名稱進行存取：

* root 是指元件的最外層元素。
* self 引用當前元素。
* parent 引用當前元素的父元素。

## 元素的定位和佈局

所有視覺元素都顯示在一個視窗中。x 和 y 屬性儲存元素相對於其父元素的座標。Slint 透過將父元素的位置添加到元素的位置來確定元素的絕對位置。如果父元素本身有一個父元素，那麼也會添加該父元素。此計算將一直持續，直到到達頂級元素。

`width` 和 `height` 屬性存儲視覺元素的大小。

可以透過兩種方式放置元素來建立整個圖形使用者介面：

* [顯式](https://docs.slint.dev/latest/docs/slint/src/language/concepts/layouting#explicit-placement) - 通過設置 `x`、`y`、`width` 和 `height` 屬性。
* [Automatically](https://docs.slint.dev/latest/docs/slint/src/language/concepts/layouting#automatic-placement-using-layouts) - 通過使用佈局元素。
  * `VerticalLayout` / `HorizontalLayout`：子項沿垂直軸或水平軸放置。
  * `GridLayout`：子項被放置在列和行的網格中。
  * 您還可以嵌套佈局以創建複雜的用戶介面。

顯式放置非常適合元素較少的靜態場景。佈局適用於複雜的使用者介面，並有助於創建可擴展的使用者介面。佈局元素表示元素之間的幾何關係。
