# cffi

## 引入libc庫

由於cffi的資料型別與rust不完全相同，我們需要引入libc庫來表達對應ffi函式中的型別。

在以前`libc`庫是和`rust`一起釋出的，後來libc被移入了`crates.io`通過cargo安裝。

```rust
// 在Cargo.toml中新增以下行
[dependencies]
libc = ">0.2.9"

// 在你的rs檔案中引入庫
extern crate libc;
```

## 宣告ffi函數

```rust
se libc::c_int;
use libc::c_void;
use libc::size_t;

#[link(name = "yourlib")]
extern {
    fn your_func(arg1: c_int, arg2: *mut c_void) -> size_t; // 宣告ffi函式
    fn your_func2(arg1: c_int, arg2: *mut c_void) -> size_t;
    static ffi_global: c_int; // 宣告ffi全域性變數
}
```

宣告一個ffi庫需要一個標記有#\[link(name = "yourlib")]的extern塊。

name為對應的庫(so/dll/dylib/a)的名字。 例如：如果你需要snappy庫(libsnappy.so/libsnappy.dll/libsnappy.dylib/libsnappy.a), 則對應的name為snappy。 在一個extern塊中你可以宣告任意多的函式和變數。

## 呼叫ffi函數

宣告完成後就可以進行呼叫了。 由於此函式來自外部的c庫，所以rust並不能保證該函式的安全性。因此，呼叫任何一個ffi函式需要一個unsafe塊。

```rust
let result: size_t = unsafe {
    your_func(1 as c_int, Box::into_raw(Box::new(3)) as *mut c_void)
};
```

## 封裝unsafe，暴露安全介面

在做c庫的rust binding時，我們做的最多的將是將不安全的c介面封裝成一個安全介面。 通常做法是：在一個叫ffi.rs之類的檔案中寫上所有的extern塊用以宣告ffi函式。在一個叫wrapper.rs之類的檔案中進行包裝：對外暴露(pub use) your\_func\_wrapper函式即可。

```rust
// ffi.rs
#[link(name = "yourlib")]
extern {
    fn your_func(arg1: c_int, arg2: *mut c_void) -> size_t;
}

// wrapper.rs
fn your_func_wrapper(arg1: i32, arg2: &mut i32) -> isize {
    unsafe { your_func(1 as c_int, Box::into_raw(Box::new(3)) as *mut c_void) } as isize
}
```

## 資料結構對應

libc為我們提供了很多原始資料型別，比如c\_int, c\_float等，但是對於自定義型別，如結構體，則需要我們自行定義。

### 結構體

rust中結構體預設的記憶體表示和c並不相容。如果要將結構體傳給ffi函式，請為rust的結構體打上標記，此外，如果使用#\[repr(C, packed)]將不為此結構體填充空位用以對齊。

```rust
#[repr(C)]
struct RustObject {
    a: c_int,
    // other members
}
```

### Union

rust到目前為止(2016-03-31)還沒有一個很好的應對c的union的方法。只能通過一些hack來實現。

### enum

和struct一樣，新增#\[repr(C)]標記即可。

### 回調函數(callback function)

和c庫打交道時，我們經常會遇到一個函式接受另一個回撥函式的情況。將一個rust函式轉變成c可執行的回撥函式非常簡單：在函式前面加上extern "C"。

```rust
extern "C" fn callback(a: c_int) { // 這個函式是傳給c呼叫的
    println!("hello {}!", a);
}

#[link(name = "yourlib")]
extern {
   fn run_callback(data: i32, cb: extern fn(i32));
}

fn main() {
    unsafe {
        run_callback(1 as i32, callback); // 列印 1
    }
}

// 對應c庫程式碼
typedef void (*rust_callback)(int32_t);

void run_callback(int32_t data, rust_callback callback) {
    callback(data); // 呼叫傳過來的回撥函式
}
```

### 字串

rust為了應對不同的情況，有很多種字串型別。其中CStr和CString是專用於ffi互動的。

### cstr

對於產生於c的字串(如在c程式中使用malloc產生)，rust使用CStr來表示，和str型別對應，表明我們並不擁有這個字串。

```rust
use std::ffi::CStr;
use libc::c_char;
#[link(name = "yourlib")]
extern {
    fn char_func() -> *mut c_char;
}

fn get_string() -> String {
    unsafe {
        let raw_string: *mut c_char = char_func();
        let cstr = CStr::from_ptr(my_string());
        cstr.to_string_lossy().into_owned()
    }
}
```

在這裡get\_string使用CStr::from\_ptr從c的char\*獲取一個字串，並且轉化成了一個String.

* 注意to\_string\_lossy()的使用：因為在rust中一切字元都是採用utf8表示的而c不是， 因此如果要將c的字串轉換到rust字串的話，需要檢查是否都為有效utf-8位元組。
* to\_string\_lossy將返回一個Cow型別， 即如果c字串都為有效utf-8位元組，則將其0開銷地轉換成一個\&str型別，若不是，rust會將其拷貝一份並且將非法位元組用U+FFFD填充。

### CString

和CStr表示從c中來，rust不擁有歸屬權的字串相反，CString表示由rust分配，用以傳給c程式的字串。

```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern {
    fn my_printer(s: *const c_char);
}

let c_to_print = CString::new("Hello, world!").unwrap();
unsafe {
    // 使用 as_ptr 將CString轉化成char指標傳給c函式
    my_printer(c_to_print.as_ptr()); 
}
```

注意c字串中並不能包含\0位元組(因為\0用來表示c字串的結束符),因此CString::new將返回一個Result， 如果輸入有\0的話則為Error(NulError)。

### 不透明結構體

C庫存在一種常見的情況：庫作者並不想讓使用者知道一個資料型別的具體內容，因此常常提供了一套工具函式，並使用void\*或不透明結構體傳入傳出進行操作。 比較典型的是ncurse庫中的WINDOW型別。

當引數是void_時，在rust中可以和c一樣，使用對應型別_mut libc::c\_void進行操作。如果引數為不透明結構體，rust中可以使用空白enum進行代替:

```rust
enum OpaqueStruct {}

extern "C" {
    pub fn foo(arg: *mut OpaqueStruct);
}

// c code
struct OpaqueStruct;
void foo(struct OpaqueStruct *arg);
```

### 空指標

另一種很常見的情況是需要一個空指標。可使用0 as \*const \_ 或者 std::ptr::null()來生產一個空指標。

## 記憶體安全

由於ffi跨越了rust邊界，rust編譯器此時無法保障程式碼的安全性，所以在涉及ffi操作時要格外注意。

### 物件解構

在涉及ffi呼叫時最常見的就是解構問題：這個物件由誰來解構？是否會洩露或use after free？ 有些情況下c庫會把一類型別malloc了以後傳出來，然後不再處理它的解構。因此在做ffi操作時請為這些型別實現解構(Drop Trait)。

### 可空指標優化

當rust的一個enum為一種特殊結構：它有兩種例項，一種為空，另一種只有一個資料域的時候，rustc會開啟空指標優化將其優化成一個指標。 比如Option\<extern "C" fn(c\_int) -> c\_int>會被優化成一個可空的函式指標。

### ownership處理

在rust中，由於編譯器會自動插入解構程式碼到塊的結束位置，在使用owned型別時要格外的注意。

```rust
extern {
    pub fn foo(arg: extern fn() -> *const c_char);
}

extern "C" fn danger() -> *const c_char {
    let cstring = CString::new("I'm a danger string").unwrap();
    cstring.as_ptr()
}  // 由於CString是owned型別，在這裡cstring被rust free掉了。USE AFTER FREE! too young!

fn main() {
  unsafe {
        foo(danger); // boom !!
    }
}
```

由於as\_ptr接受一個\&self作為引數(fn as\_ptr(\&self) -> \*const c\_char)，as\_ptr以後ownership仍然歸rust所有。因此rust會在函式退出時進行析構。 正確的做法是使用into\_raw()來代替as\_ptr()。由於into\_raw的簽名為fn into\_raw(self) -> \*mut c\_char，接受的是self,產生了ownership轉移， 因此danger函式就不會將cstring解構了。

### panic

由於在ffi中panic是未定義行為，切忌在cffi時panic包括直接呼叫panic!,unimplemented!,以及強行unwrap等情況。

## 靜態庫/動態庫

前面提到了宣告一個外部庫的方式--#\[link]標記，此標記預設為動態庫。但如果是靜態庫，可以使用#\[link(name = "foo", kind = "static")]來標記。 此外，對於osx的一種特殊庫--framework, 還可以這樣標記#\[link(name = "CoreFoundation", kind = "framework")].

## 呼叫約定(call conventation)

前面看到，宣告一個被c呼叫的函式時，採用extern "C" fn的語法。此處的"C"即為c呼叫約定的意思。此外，rust還支援：

* stdcall
* aapcs
* cdecl
* fastcall
* vectorcall //這種call約定暫時需要開啟abi\_vectorcall feature gate.
* Rust
* rust-intrinsic
* system
* C
* win64
