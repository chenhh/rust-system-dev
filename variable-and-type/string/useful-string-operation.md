# 常用字串操作

## 印出16進位

[fmt syntax](https://doc.rust-lang.org/std/fmt/index.html)。

[u32::from\_str\_radix](https://doc.rust-lang.org/std/primitive.u32.html#method.from\_str\_radix)。從str讀取數值，指定基數後，轉為u32類型之值。

```rust
use std::u32;

fn to_hex(val: &str, len: usize) -> String {
    let n: u32 = u32::from_str_radix(val, 2).unwrap();
    //  01$x essentially means that the number must be printed 
    // in hexadecimal form padded with zeros up to the length 
    // passed as the second (1st with zero-based indexing) argument.
    format!("{:01$x}", n, len * 2)
}

fn main() {
    println!("{}", to_hex("110011111111101010110010000100", 6))
    // prints 000033feac84
}
```

## Conversion between String, str, Vec\<u8>, Vec\<char>

[https://gist.github.com/jimmychu0807/9a89355e642afad0d2aeda52e6ad2424](https://gist.github.com/jimmychu0807/9a89355e642afad0d2aeda52e6ad2424)

```rust
use std::str;

fn main() {
  // -- FROM: vec of chars --
  let src1: Vec<char> = vec!['j','{','"','i','m','m','y','"','}'];
  // to String
  let string1: String = src1.iter().collect::<String>();
  // to str
  let str1: &str = &src1.iter().collect::<String>();
  // to vec of byte
  let byte1: Vec<u8> = src1.iter().map(|c| *c as u8).collect::<Vec<_>>();
  println!("Vec<char>:{:?} | String:{:?}, str:{:?}, Vec<u8>:{:?}", src1, string1, str1, byte1);

  // -- FROM: vec of bytes --
  // in rust, this is a slice
  // b - byte, r - raw string, br - byte of raw string
  let src2: Vec<u8> = br#"e{"ddie"}"#.to_vec();
  // to String
  // from_utf8 consume the vector of bytes
  let string2: String = String::from_utf8(src2.clone()).unwrap();
  // to str
  let str2: &str = str::from_utf8(&src2).unwrap();
  // to vec of chars
  let char2: Vec<char> = src2.iter().map(|b| *b as char).collect::<Vec<_>>();
  println!("Vec<u8>:{:?} | String:{:?}, str:{:?}, Vec<char>:{:?}", src2, string2, str2, char2);

  // -- FROM: String --
  let src3: String = String::from(r#"o{"livia"}"#);
  let str3: &str = &src3;
  let char3: Vec<char> = src3.chars().collect::<Vec<_>>();
  let byte3: Vec<u8> = src3.as_bytes().to_vec();
  println!("String:{:?} | str:{:?}, Vec<char>:{:?}, Vec<u8>:{:?}", src3, str3, char3, byte3);

  // -- FROM: str --
  let src4: &str = r#"g{'race'}"#;
  let string4 = String::from(src4);
  let char4: Vec<char> = src4.chars().collect();
  let byte4: Vec<u8> = src4.as_bytes().to_vec();
  println!("str:{:?} | String:{:?}, Vec<char>:{:?}, Vec<u8>:{:?}", src4, string4, char4, byte4);
```

