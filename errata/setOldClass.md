
S3クラスをS4メソッドのシグネチャとして使うときの注意
====================================================

S3のクラスは、S4のシグネチャとして一見うまく動作します。例えば、`Dummy`というS4クラスと、`super-data-frame`という`data.frame`を継承したS3クラスがあるとします。

``` r
setClass("Dummy", slots = c("x"))

# コンストラクタ
as_super_data_frame <- function(df) {
  class(df) <- c("super-data-frame", class(df))
  df
}
```

この2つをシグネチャとして持つ`doSomething()`というメソッドを考えます。`super-data-frame`はS4のクラスとしては登録されていないので定義時に警告は出ますが、実行時には正しいメソッドがディスパッチされています。

``` r
setGeneric("doSomething",
           function(dummy, df) standardGeneric("doSomething"))
#> [1] "doSomething"

setMethod("doSomething",
          signature = c("Dummy", "super-data-frame"),
          function(dummy, df) print("yeah!"))
#> in method for 'doSomething' with signature '"Dummy","super-data-frame"': no definition for class "super-data-frame"
#> [1] "doSomething"

dummy <- new("Dummy", x = "foo")
super_iris <- as_super_data_frame(iris)

doSomething(dummy, super_iris)
#> [1] "yeah!"
```

しかし、S3クラスの継承関係はS4に自動では引き継がれません。

今度は、シグネチャとして`super-data-frame`ではなく`data.frame`を持つ次の`doSomething2()`というメソッドを考えてみましょう。

``` r
setGeneric("doSomething2",
           function(dummy, df) standardGeneric("doSomething2"))
#> [1] "doSomething2"

setMethod("doSomething2",
          signature = c("Dummy", "data.frame"),
          function(dummy, df) print("yeah! yeah!"))
#> [1] "doSomething2"

doSomething2(dummy, super_iris)
#> Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'doSomething2' for signature '"Dummy", "super-data-frame"'
```

今度はエラーになってしまいました。これは、S4のクラスシステムからは`super-data-frame`が`data.frame`を継承したクラスだとは分からないからです。この継承関係をS4に持ち込むには、`setOldClass()`であらためて継承関係を登録する必要があります。

``` r
setOldClass(c("super-data-frame", "data.frame"))
```

もう一度同じ`doSomething2()`を実行してみると、今度はエラーにはなりません。

``` r
doSomething2(dummy, iris)
#> [1] "yeah! yeah!"
```

実は、これと同じようなバグが初期のtibbleパッケージにはありました。「Learning R Programming」はそのバグがまだ修正されていないときに書かれたため、`tbl_df`クラスのオブジェクトを`dbWriteTable()`に指定するとき、

``` r
dbWriteTable(con, "diamonds", diamonds, row.names = FALSE)
```

ではなく、

``` r
dbWriteTable(con, "diamonds", as.data.frame(diamonds), row.names = FALSE)
```

と指定するように指示されています（319ページ）。しかし、このバグはtibbleパッケージのバージョン1.0.0で修正されたため、現在は`as.data.frame()`で変換しなくてもエラーにはなりません。
