---
title: "たった1日で基本が身に付く！ Go言語 超入門@6日目"
date: 2020-11-30T22:56:48+09:00
draft: false
---

[たった1日で基本が身に付く！ Go言語 超入門：書籍案内｜技術評論社](https://gihyo.jp/book/2020/978-4-297-11617-0)

旅行めちゃくちゃよかったです！
また行くために仕事も勉強も頑張ります。

## CHAPTER 7　Goプログラミングの注意事項
### Go言語のnilの使う場合
* ほとんどの場合は以下の3通りと考えて問題ない
  * ①値が代入されていない変数のポインタ表現
  * ②空のスライス
  * error型

### フィールドの値を入れないとどうなるか
```
type person struct {
  name string
  age int
  smokes bool
  friend *person
}

myperson := person{}
fmt.Println(myperson.name)   // ""
fmt.Println(myperson.age)    // 0
fmt.Println(myperson.bool)   // false
fmt.Println(myperson.friend) // <nil>
```

* データ型が分かっていても値が定義されていない場合は、自動的に「初期値」が入る

* 文字列(string)型
  * 空の文字列
* 数値型
  * 整数、少数など、データ型に応じた0値
* bool型
  * false

### 変数を宣言したままの場合
```
var testint int
var teststring string

fmt.Println(testint)    // 0
fmt.Println(teststring) // ""
```

* 構造体を変数の型にした場合はどうなる?
```
type mynumber struct {
  value int
}

var somenumber mynumber
fmt.Println(somenumber) // {0}
```
* フィールドvalueに初期値0が入った構造体が自動で作成される。

### 配列とスライスで要素を決めない場合
```
var somearr [5]int

fmt.Println(somearr) // [0 0 0 0 0]
```
* 5つの要素にすべて初期値0が与えられた配列が自動で作成される。

```
var somesl []int

fmt.Println(somesl) // []
```
* 自動で作成されているが値がnilかはわからない

```
var somesl []int

if somesl == nil {
  fmt.Println("空のスライスはnil") // 空のスライスはnil
}
```
* 空のスライスはnil

### 整数を文字列に変換するには
* 整数などの値から文字列を作り上げるには、fmt.Sprintf関数を使う
```
a := 9 * 9
spf := fmt.Sprintf("9 * 9は%d", a)
smt.Println(spf)
```

* 文字列と整数は結合できない
* 解決方法は、整数を文字列に変換すること
```
// strconvパッケージのインポート
import (
  "fmt"
  "strconv"
)

a := 9 * 9
astr := "9の2乗も" + strconv.Itoa(a)
fmt.Println(astr)
```

### 少数を文字列に変換するには
```
// 第1引数：出力した数値が代入された変数
// 第2引数：出力書式。シングルクォーテーションで囲むこと
// 第3引数：小数点以下の桁数
// 第4引数：float32かfloat64か
bstr := strconv.FormatFloat(b, 'f', 2, 64)
```

### 整数と少数を相互に変換

```
// ①
a := 5 * 2.5
fmt.Println(a) // 12.5

// ②
b := 6
c := b + 0.2 // エラー
fmt.Println(c)
```
* ①ではエラーにならない
  * aには少数が入る。`:=`では型を自動で判別するため少数と認識されるのでエラーにならない
* ②ではエラーになる
  * bは整数として認識される。
  * 次にb(整数)と0.2(少数)を混ぜて計算できないのでエラーになる。

* **大事な事は整数と少数を混ぜないこと**

```
b := 6
c := float64(b) + 0.2 // エラーにならない
fmt.Println(c)
```

### 変数の値のデータ型を確認する
* `%T`にして出力する
```
fmt.Printf("aのデータ型は%T¥n", a)
fmt.Printf("bのデータ型は%T¥n", b)
fmt.Printf("cのデータ型は%T¥n", c)
```

### 結果としてエラーインスタンスをともに戻す関数
* Atoi(ASCII to Integer)関数の使い方
```
nm, err := strconv.Atoi("123")
```
* 戻り値を2つ返す
* nm
  * 変換した結果の整数
* err
  * error型というデータ型のインスタンス
  * 正常に実行されればnilが与えられる
  * 一方エラーであれば、nmには0が代入される

### Atoiが返すerror型インスタンス
* Atoi関数が結果とともにerrorを返すのは、errorがnil出なかったら適切な処理をするため
```
nm, err := strconv.Atoi("Waht")
fmt.Println(err.Error()) // strconv.Atoi: parsing "What": invalid syntax
```
* このようにGo言語に備わったパッケージの関数の中には、処理結果とerrorを返すというものが多くある

### errorの正体は「インターフェイス」
* errorはGo言語に標準で実装されているデータ型
* その正体はerrorインターフェイス

```
type error interface {
  Error() string
}
```

### 整数の細かいデータ型
* int8、int16、int32
  * メモリを無駄に使いたくないとき、あえて使うことがある
* uint8、uint16、uint32、uint64
  * uはunsigned(符号なし)の意味で、負でないことが判明し、なるべるメモリを割り当てたいときに使用する
  * uintと書くと、64ビットシステムではuint64が割り当てられる。
* uintptr
  * ポインタのビット表現を保持する

### 複素数
* 64ビットシステムでは、complex128
* 32ビットシステムでは、complex64

### 文字列の解析
* Go言語では、文字列を数値に変換して解析できる。

```
package main
import (
  "fmt"
)
func main() {
  str := "internationalzation"
  strja := "達磨さんが転んだ"
  bt := []byte(str) // 英数字をbyte変換(byteはuint8の別名)
  fmt.Println(bt)
  rn := []rune(strja) // 日本語をrune変換(runeはint32の別名)
  fmt.Println(rn)
}
```

* 実行結果
```
[105 110 116 101 114 110 97 116 105 111 110 97 108 122 97 116 105 111 110]
[36948 30952 12373 12435 12364 36578 12435 12384]
```

* 数値から文字列に戻す作業
```
strback := ""
for i := 0; i < len(bt); i++ {
  strback += string(bt[i])
}
strbackja := ""
for i := 0; i < len(rn); i++ {
  strbackja += string(rn[i])
}
fmt.Println(strback)
fmt.Println(strbackja)
}
```

* 実行結果
```
internationalzation
達磨さんが転んだ
```

### mapとrange
* mapは「キー」と値の1対1対応の集合
```
legs := map[string]int{
  "bird":        2,
  "cat":         4,
  "grasshopper": 6,
  "octopus":     8,
  "squid":       10,
}
```
* `map[キー]値`のように宣言
* キーと値の間はコロン「:」で区切る
* 配列同様に{}で囲む

* mapの要素を順に取り出すにはrangeというキーワードで繰り返し処理を行う。

```
for k, v := range legs {
  fmt.Printf("key: %s, value: %d\n", k, v)
}
```

## 今日の学び
* 実践で多く出てきそうな内容だった。
* 駆け足でやったので早く実戦形式で使う時やハマったときに参照して覚えていきたい。
