---
title: "Golang 练习N则"
date: 2020-01-29T01:45:09+08:00
categories: ['Code']
tags: ['Exercise', 'Golang']
author: "Fancy"

resizeImages: false
---
golang练手项目，一周每天做两个Golang练习，近期温习一下并适当做些注释
<!--more-->

在主Python的时候，也要巩固提升自己的Go姿势水平，练习题目来自[A Tour of Go](https://go-tour-zh-tw.appspot.com)，提供多语言，教程说实话比较经典了，闲的时候值得一看的Golang初中级进阶教程，比想象中的要闹心。干拉会很吃力，刚好查漏补缺，还是需要其他Golang基础的。




## 練習：循環和函式

作為練習函式和循環的簡單途徑，用牛頓法實現開方函式。

在這個例子中，牛頓法是通過選擇一個初始點 *z* 然後重複這一過程求 `Sqrt(x)` 的近似值：

![img](https://go-tour-zh-tw.appspot.com/content/img/newton.png)

為了做到這個，只需要重複計算 10 次，並且觀察不同的值（1，2，3，……）是如何逐步逼近結果的。 然後，修改循環條件，使得當值停止改變（或改變非常小）的時候退出循環。觀察迭代次數是否變化。結果與 [math.Sqrt](http://golang.org/pkg/math/#Sqrt) 接近嗎？

提示：定義並初始化一個浮點值，向其提供一個浮點語法或使用轉換：

```go
z := float64(1)
z := 1.0
```

```go
package main

import (
    "fmt"
    "math"
)

func Sqrt(x float64) float64 {
    z, k := float64(1), float64(1)
    for k > 1e-15 {
        d := z
        z = z - (z*z-x)/(2*z)
        k = z - d
        if k < 0 {
            k = d - z
        }
    }
    return z
}

func main() {
    fmt.Println(Sqrt(2))
    fmt.Println(math.Sqrt(2))
}
```





## 練習：slice

實現 `Pic` 。它返回一個 slice 的長度 `dy` ，和 slice 中每個元素的長度的 8 位無符號整數 `dx` 。當執行這個程式，它會將整數轉換為灰度（好吧，藍度）圖片進行展示。

圖片的實現已經完成。可能用到的函式包括 `>x^y` ， `(x+y)/2` 和 `x*y` 。

（需要使用循環來分配 `[][]uint8` 中的每個 `[]uint8`。）

（使用 `uint8(intValue)` 在類型之間進行轉換。）





```go
package main

import "code.google.com/p/go-tour/pic"

func Pic(dx, dy int) [][]uint8 {
    img := make([][]uint8, dy)
    for x := 0; x < dy; x++ {
        img[x] = make([]uint8, dx)
        for y := 0; y < dx; y++ {
            img[x][y] = uint8((x ^ y) * y)
        }
    }
    return img
}

func main() {
    pic.Show(Pic)
}
```



---

## 練習：map

實現 `WordCount` 。它應當返回一個含有 `s` 中每個「詞」個數的 map。函式 `wc.Test` 針對這個函式執行一個測試用例，並顯示成功或者失敗。

你會發現 [strings.Fields](http://golang.org/pkg/strings/#Fields) 很有幫助。



```go
package main

import (
    "code.google.com/p/go-tour/wc"
    "strings"
)

func WordCount(s string) map[string]int {
    source := make(map[string]int)
  
  // 將字符串分成單個word
    words := strings.Fields(s)
  
  // 遍歷單詞，出現`s`則+1
    for _, word := range words {
        source[word]++
    }
    return source
}

func main() {
    wc.Test(WordCount)
}
```

## 練習：斐波納契閉包

現在來通過函式做些有趣的事情。

實現一個 `fibonacci` 函式，返回一個函式（一個閉包）可以返回連續的斐波納契數。

```go
package main

import "fmt"

// fibonacci is a function that returns
// a function that returns an int.
func fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

func main() {
    f := fibonacci()
    for i := 0; i < 10; i++ {
        fmt.Println(f())
    }
}
```

---

## 進階練習：複數的立方跟

讓我們通過 `complex64` 和 `complex128` 來探索一下 Go 內建的複數。對於立方根，牛頓法需要大量循環：

![img](https://go-tour-zh-tw.appspot.com/content/img/newton3.png)

找到 2 的立方根，確保算法能夠工作。在 `math/cmplx` 包中有 [Pow](http://golang.org/pkg/math/cmplx/#Pow) 函式。

```go
package main

import (
    "fmt"
    "math/cmplx"
)

func Cbrt(x complex128) complex128 {
    z := complex128(1)
    for i := 0; i < 1000; i++ {
        z = z - (cmplx.Pow(z, complex128(3))-x)/(3*cmplx.Pow(z, complex128(2)))
    }
    return z
}

func main() {
    fmt.Println(Cbrt(2))
}
```

## 練習：錯誤

​    從之前的練習中複製 `Sqrt` 函式，並修改使其返回 `error` 值。     

​    `Sqrt` 接收到一個負數時，應當返回一個非 nil 的錯誤值。複數同樣也不被支持。  

​    創建一個新類型  

```go
type ErrNegativeSqrt float64
```

​    為其實現  

```go
func (e ErrNegativeSqrt) Error() string
```

​    使其成為一個 `error`， 該方法就可以讓 `ErrNegativeSqrt(-2).Error()` 返回 `"cannot Sqrt negative number: -2"`。  

​    **注意：** 在 `Error` 方法內調用 `fmt.Print(e)` 將會讓程式陷入死循環。可以通過先轉換 `e` 來避免這個問題：`fmt.Print(float64(e))`。請思考這是為什麼呢？  

​    修改 `Sqrt` 函式，使其接受一個負數時，返回 `ErrNegativeSqrt` 值。  

#### 解答：

```go
package main

import (
    "fmt"
    "math"
)

type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
    return fmt.Sprintf("cannot Sqrt negative number: %v", float64(e))
}

func Sqrt(x float64) (float64, error) {
    if x < 0 {
        return 0, ErrNegativeSqrt(x)
    }
    z := x / 2.0
    for {
        y := z - (z*z-x)/(2*z)
        if math.Abs(y-z) < 1e-9 {
            return y, nil
        }
        z = y
    }
    return z, nil
}

func main() {
    fmt.Println(Sqrt(2))
    fmt.Println(Sqrt(-2))
}
```

---

## 練習：HTTP 處理

​    實現下面的類型，並在其上定義 ServeHTTP 方法。在 web 服務器中註冊它們來處理指定的路徑。  

```go
type String string

type Struct struct {
    Greeting string
    Punct    string
    Who      string
}
```

​    例如，可以使用如下方式註冊處理方法：  

```go
http.Handle("/string", String("I'm a frayed knot."))
http.Handle("/struct", &Struct{"Hello", ":", "Gophers!"})
```

#### 解答：

```go
type String string

func (s String) ServeHTTP(
    w http.ResponseWriter,
    r *http.Request) {
    fmt.Fprint(w, "%s", s)
}

type Struct struct {
    Greeting string
    Punct    string
    Who      string
}

func (s Struct) ServeHTTP(
    w http.ResponseWriter,
    r *http.Request) {
    fmt.Fprint(w, s.Greeting+s.Punct+s.Who)
}

func main() {
    http.Handle("/string", String("I'm a frayed knot."))
    http.Handle("/struct", &Struct{"Hello", ":", "Gophers!"})
    http.ListenAndServe("localhost:4000", nil)
}
```

## 練習：圖片

​    還記得之前編寫的圖片生成器嗎？現在來另外編寫一個，不過這次將會返回 `image.Image` 來代替 slice 的數據。  

​    自定義的 `Image` 類型，要實現[必要的方法](http://golang.org/pkg/image/#Image)，並且調用 `pic.ShowImage` 。  

​    `Bounds` 應當返回一個 `image.Rectangle`，例如 `image.Rect(0, 0, w, h)` 。  

​    `ColorModel` 應當返回 `image.RGBAModel` 。  

​    `At` 應當返回一個顏色；在這個例子裡，在最後一個圖片生成器的值 `v` 匹配 `color.RGBA{v, v, 255, 255}` 。  

```go
package main

import (
    "code.google.com/p/go-tour/pic"
    "image"
    "image/color"
)

type Image struct {
    w, h int
    seed uint8
}

func (self *Image) ColorModel() color.Model {
    return color.RGBAModel
}
func (self *Image) Bounds() image.Rectangle {
    return image.Rect(0, 0, self.w, self.h)
}
func (self *Image) At(x, y int) color.Color {
    return color.RGBA{self.seed + uint8(x), self.seed + uint8(y), 255, 255}
}

func main() {
    m := Image{100, 100, 120}
    pic.ShowImage(&m)
}
```

---

## 練習：Rot13 讀取器

​    一般的模式是 [io.Reader](http://golang.org/pkg/io/#Reader) 包裹另一個 `io.Reader` ，用某些途徑修改特定的流。  

​    例如，[gzip.NewReader](http://golang.org/pkg/compress/gzip/#NewReader) 函式輸入一個 `io.Reader` （gzip 的數據流）並且返回一個同樣實現了 `io.Reader` 的 `*gzip.Reader`（解壓縮後的數據流）。  

​    實現一個實現了 `io.Reader` 的 `rot13Reader` ，用 [ROT13](http://en.wikipedia.org/wiki/ROT13) 修改數據流中的所有的字母進行密文替換。  

​    `rot13Reader` 已經提供。通過實現其 `Read` 方法使得它匹配 `io.Reader` 。  

```go
package main

import (
    "io"
    "os"
    "strings"
)

type rot13Reader struct {
    r io.Reader
}

func (self *rot13Reader) Read(p []byte) (int, error) {
    self.r.Read(p)
    leng := len(p)
    for i := 0; i < leng; i++ {
        switch {
        case p[i] >= 'a' && p[i] < 'n':
            fallthrough
        case p[i] >= 'A' && p[i] < 'N':
            p[i] = p[i] + 13
        case p[i] >= 'n' && p[i] <= 'z':
            fallthrough
        case p[i] >= 'N' && p[i] <= 'Z':
            p[i] = p[i] - 13
        }
    }
    return leng, nil
}

func main() {
    s := strings.NewReader(
        "Lbh penpxrq gur pbqr!")
    r := rot13Reader{s}
    io.Copy(os.Stdout, &r)
}
```
