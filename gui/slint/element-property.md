# 元件屬性

{% embed url="https://docs.slint.dev/latest/docs/slint/src/language/syntax/properties" %}

```java
export component Example inherits Window {
    // Simple expression: ends with a semi colon
    width: 42px;
    // or a code block (no semicolon needed)
    height: { 42px }
}
```

所有元素都有屬性。內置元素具有常見屬性，例如顏色或維度屬性。您可以為它們配置值或整個[表示式](https://docs.slint.dev/latest/docs/slint/src/language/syntax/expressions.html)。

屬性的預設值是類型的預設值。例如，布爾屬性預設為 `false，int` 屬性預設為零，等等。

除了現有屬性之外，還可以透過指定類型、名稱和可選的預設值來定義額外的屬性。
