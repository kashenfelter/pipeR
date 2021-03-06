

# pipeR

[![Linux Build Status](https://travis-ci.org/renkun-ken/pipeR.png?branch=master)](https://travis-ci.org/renkun-ken/pipeR) 
[![Windows Build status](https://ci.appveyor.com/api/projects/status/github/renkun-ken/pipeR?svg=true)](https://ci.appveyor.com/project/renkun-ken/pipeR)
[![codecov.io](http://codecov.io/github/renkun-ken/pipeR/coverage.svg?branch=master)](http://codecov.io/github/renkun-ken/pipeR?branch=master)
[![CRAN Version](http://www.r-pkg.org/badges/version/pipeR)](https://cran.r-project.org/package=pipeR)

pipeR は、関数を連鎖的に適用する方法を、様々なスタイルで提供します：

* パイプ演算子
* パイプオブジェクト
* pipeline 関数

これらは、それぞれ異なる方法でパイプラインを実現していますが、機能としてほぼ同じものを提供します。ある値を次の演算にパイプするときに、

* 関数の最初の引数として渡す
* 表現式中のドット記号(`.`)として渡す
* モデル式(formula)によって定義された変数として渡す
* 入力を次へ持ち越しながら、副作用を起こす
* パイプラインの途中で生成された値を変数に代入(保存)する

といったことができます。  

pipeR の文法は、パイプラインの読みやすさと、広く様々な演算との親和性を考えて設計されています。

**pipeR の完全なガイドとしては、[pipeR Tutorial](http://renkun.me/pipeR-tutorial) を強くお勧めします。**

## インストール

最新の開発バージョンを GitHub からインストールするには：

```r
devtools::install_github("renkun-ken/pipeR")
```

[CRAN](https://cran.r-project.org/package=pipeR) からインストールするには：

```r
install.packages("pipeR")
```

## はじめよう

次のコードは従来のアプローチで書かれたものです：

```r
plot(density(sample(mtcars$mpg, size = 10000, replace = TRUE), 
  kernel = "gaussian"), col = "red", main="density of mpg (bootstrap)")
```

R の組み込みデータセットである `mtcars` の変数 `mpg` に対して、ブートストラップ法でリサンプリングし、ガウシアンカーネルを用いて推定した密度関数をプロットしています。

このコードは深くネストされているため、読むのもコードをメンテナンスするのも一苦労です。次のコードは、この従来のコードを、パイプ演算子、`Pipe()` 関数、`pipeline()` 関数で書き直したものです。

* 演算子によるパイプライン

```r
mtcars$mpg %>>%
  sample(size = 10000, replace = TRUE) %>>%
  density(kernel = "gaussian") %>>%
  plot(col = "red", main = "density of mpg (bootstrap)")
```

* オブジェクトによるパイプライン(`Pipe()`)

```r
Pipe(mtcars$mpg)$
  sample(size = 10000, replace = TRUE)$
  density(kernel = "gaussian")$
  plot(col = "red", main = "density of mpg (bootstrap)")
```

* 引数によるパイプライン

```r
pipeline(mtcars$mpg,
  sample(size = 10000, replace = TRUE),
  density(kernel = "gaussian"),
  plot(col = "red", main = "density of mpg (bootstrap)"))
```

* 表現式によるパイプライン

```r
pipeline({
  mtcars$mpg
  sample(size = 10000, replace = TRUE)
  density(kernel = "gaussian")
  plot(col = "red", main = "density of mpg (bootstrap)")  
})
```

## 使い方

### `%>>%`

パイプ演算子 `%>>%` は、基本的に、左辺の値を右辺の計算式に渡します。右辺の計算式は、その文法に従って評価されます。

#### 関数の最初の引数として渡す

多くの R 関数は、データを最初の引数として受け取るため、パイプとの親和性が高いです。このような引数配置であれば、パイプにより操作の流れを作ることができます。つまり、一つのデータソースを関数の最初の引数として入力し、関数を適用し、次の関数の最初の引数として渡すことができます。このようなコマンドの連鎖をパイプラインと呼びます。

`%>>%` の右側が、関数名か関数呼び出しである場合、左側の値は常にその関数の最初の引数になります。

```r
rnorm(100) %>>%
  plot
```

```r
rnorm(100) %>>%
  plot(col="red")
```

左辺の値が複数の箇所で必要となる場合もあります。その場合、関数呼び出しの中で `.` を使うことで表現することができます。

```r
rnorm(100) %>>%
  plot(col="red", main=length(.))
```

関数を名前空間を指定して呼び出す場合、関数の終わりには必ず `()` をつけて呼び出して下さい。

```r
rnorm(100) %>>%
  stats::median()
  
rnorm(100) %>>%
  graphics::plot(col = "red")
```

#### 表現式中の `.` にパイプする 

R の全ての関数がパイプに適合するとは限りません。いくつかの関数は、データを最初の引数として受け取りません。そのような場合には、関数呼び出しを `{}` もしくは `()` で囲み、表現式にしてから `.` を使ってパイプするとよいでしょう。

```r
mtcars %>>%
  { lm(mpg ~ cyl + wt, data = .) }
```

```r
mtcars %>>%
  ( lm(mpg ~ cyl + wt, data = .) )
```

#### モデル式(formula)を用いたラムダ式でパイプする

`.` を使ったパイプでは、時々混乱が起こります。例えば：

```r
mtcars %>>%
  (lm(mpg ~ ., data = .))
```

これはちゃんと動きますが、二つの `.` が別の意味を持っているため、紛らわしく感じます。

このような場合、pipeR ではラムダ式を使うことができます。ラムダ式は `()` の中にモデル式(formula)を書くことで表現されます。例えば、`(x ~ f(x))` というラムダ式を書けば、パイプで受け取った値を変数 `x` に入れ、関数 `f(x)` を評価して次に受け渡すという意味になります。

```r
mtcars %>>%
  (df ~ lm(mpg ~ ., data = df))
```

```r
mtcars %>>%
  subset(select = c(mpg, wt, cyl)) %>>%
  (x ~ plot(mpg ~ ., data = x))
```

#### 副作用をパイプに含める

パイプラインでは、最終的な結果だけを必要とするのではなく、中間結果も出したい場合があります。中間結果をプリントしたり、プロットしたり、保存したりするためには、メインストリームのパイプラインを壊さずに、副作用を持たせる必要があります。例えば、散布図を描くために `plot()` を呼び出すと、結果として `NULL` が返されます。もし、パイプラインの中で何の工夫もなく `plot()` を呼び出せば、次に渡すべき結果が `NULL` になってしまい、パイプラインが壊れてしまいます。

パイプラインを壊さずに副作用を持たせるには、`~` で始まる片側モデル式 `(~ f(x))` を使います。これにより、右辺の式 `f(x)` は副作用のために評価されるだけであり、その結果は無視され、もともと入力された値が結果としてパイプラインに渡されるという意味になります。

```r
mtcars %>>%
  subset(mpg >= quantile(mpg, 0.05) & mpg <= quantile(mpg, 0.95)) %>>%
  (~ cat("rows:",nrow(.),"\n")) %>>%   # cat() はNULL を返す
  summary
```

```r
mtcars %>>%
  subset(mpg >= quantile(mpg, 0.05) & mpg <= quantile(mpg, 0.95)) %>>%
  (~ plot(mpg ~ wt, data = .)) %>>%    # plot() は NULL を返す
  (lm(mpg ~ wt, data = .)) %>>%
  summary()
```

`~` によって、メインストリームパイプラインと副作用操作を容易に区別することができます。

中間結果をプリントする糖衣構文も用意されています。`(? expr)` は `(~ print(expr))` と同様の動きをします。

```r
mtcars %>>% 
  (? ncol(.)) %>>%
  summary
```

#### 代入ありパイプ

プリントやプロットに加えて、中間結果を変数に代入したい場合もあります。

パイプした結果を変数 `symbol` に代入したい場合は、`(~ symbol)` と書けばいいだけです。そうすることで、現在の環境中の `symbol` という変数に、パイプで渡された値を代入することになります。
 
```r
mtcars %>>%
  (lm(formula = mpg ~ wt + cyl, data = .)) %>>%
  (~ lm_mtcars) %>>%
  summary
```

パイプした値をそのまま変数に代入するのではなく、何か変換をかけてから代入したい場合は、`=` や `<-` が使えます。より自然に、ラムダ式風に書くには、`->` を使います(提案してくれた @yanlinlin82 ありがとう)。

```r
mtcars %>>%
  (~ summ = summary(.)) %>>%  # 副作用 代入
  (lm(formula = mpg ~ wt + cyl, data = .)) %>>%
  (~ lm_mtcars) %>>%
  summary
```

```r
mtcars %>>%
  (~ summary(.) -> summ) %>>%
  
mtcars %>>%
  (~ summ <- summary(.)) %>>%
```

途中の結果を変数に代入しつつ、パイプで次に送るには、単に `(symbol = expression)` のように書けばできます：

```r
mtcars %>>%
  (~ summ = summary(.)) %>>%  # 副作用 代入
  (lm_mtcars = lm(formula = mpg ~ wt + cyl, data = .)) %>>%  # パイプを続ける
  summary
```

もしくは `(expression -> symbol)` と書けばより自然です：

```r
mtcars %>>%
  (~ summary(.) -> summ) %>>%  # side-effect assignment
  (lm(formula = mpg ~ wt + cyl, data = .) -> lm_mtcars) %>>%  # パイプを続ける
  summary
```

#### オブジェクトから要素を抽出する

`x %>>% (y)` と書けば、オブジェクト `x` から要素 `y` を抜き出すことができます。`x` としては、ベクトルでもリストでも環境でも、`[[]]` が定義されているものならなんでも OK です。さらには S4 オブジェクトでも OK です。

```r
mtcars %>>%
  (lm(mpg ~ wt + cyl, data = .)) %>>%
  (~ lm_mtcars) %>>%
  summary %>>%
  (r.squared)
```

#### 互換性

* [dplyr](https://github.com/hadley/dplyr/) と一緒に使えます：

```r
library(dplyr)
mtcars %>>%
  filter(mpg <= mean(mpg)) %>>%  
  select(mpg, wt, cyl) %>>%
  (~ plot(.)) %>>%
  (model = lm(mpg ~ wt + cyl, data = .)) %>>%
  (summ = summary(.)) %>>%
  (coefficients)
```

* [ggvis](http://ggvis.rstudio.com/) と一緒に使えます：

```r
library(ggvis)
mtcars %>>%
  ggvis(~mpg, ~wt) %>>%
  layer_points()
```

* [rlist](http://renkun.me/rlist/) と一緒に使えます：

```r
library(rlist)
1:100 %>>%
  list.group(. %% 3) %>>%
  list.mapv(g ~ mean(g))
```

### `Pipe()`

`Pipe()` 関数はパイプオブジェクトを作成します。パイプオブジェクトでは、外部演算子を使わない軽量チェインが使えます。これによるパイプラインは、典型的には `Pipe()` で始まり、`$value` または `[]` によりパイプの最終的な値を取り出すことで終わります。

パイプオブジェクトは内部関数 `.(...)` を提供します。この関数は、`x %>>% (...)` と全く同じ動きをします。また、`%>>%` には無い機能を持っています。

> 注： `.()` は `=` による代入をサポートしていません。`~`, `<-`, `->` はサポートしています。

#### パイプ

```r
Pipe(rnorm(1000))$
  density(kernel = "cosine")$
  plot(col = "blue")
```

```r
Pipe(mtcars)$
  .(mpg)$
  summary()
```

```r
Pipe(mtcars)$
  .(~ summary(.) -> summ)$
  lm(formula = mpg ~ wt + cyl)$
  summary()$
  .(coefficients)
```

#### サブセットと抽出

```r
pmtcars <- Pipe(mtcars)
pmtcars[c("mpg","wt")]$
  lm(formula = mpg ~ wt)$
  summary()
pmtcars[["mpg"]]$mean()
```

#### 値の代入

```r
plist <- Pipe(list(a=1,b=2))
plist$a <- 0
plist$b <- NULL
```

#### 副作用

```r
Pipe(mtcars)$
  .(? ncol(.))$
  .(~ plot(mpg ~ ., data = .))$    # 副作用： plot
  lm(formula = mpg ~ .)$
  .(~ lm_mtcars)$                  # 副作用： 代入
  summary()
```

#### 互換性

* dplyr と一緒に使えます：

```r
Pipe(mtcars)$
  filter(mpg >= mean(mpg))$
  select(mpg, wt, cyl)$
  lm(formula = mpg ~ wt + cyl)$
  summary()$
  .(coefficients)$
  value
```

* ggvis と一緒に使えます：

```r
Pipe(mtcars)$
  ggvis(~ mpg, ~ wt)$
  layer_points()
```

* rlist と一緒に使えます：

```r
Pipe(1:100)$
  list.group(. %% 3)$
  list.mapv(g ~ mean(g))$
  value
```

### `pipeline()`

`pipeline()` 関数は、引数ベースおよび表現式ベースのパイプライン評価メカニズムを提供します。`pipeline()` 関数の動作は、引数をどのように与えるかによって変わります。最初の引数のみを与えた場合、その引数は、`{}` で囲まれた表現式である必要があり、各行がパイプラインの 1 ステップであるとみなされます。複数の引数が与えられた場合、各引数がパイプラインの 1 ステップであるとみなされます。全てのパイプラインステップは、最終的に `%>>%` によって結合されるため、どちらの動作も全く同じものになります。

`pipeline()` 関数の引数や表現式において、注目すべき点があります。パイプライン上で特別な仕事(例えば副作用)を行うためのステップは、もはや `()` で囲む必要はありません。これは、`%>>%` を用いた場合には問題となる、演算子の優先順位について考える必要がなくなるためです。

```r
pipeline({
  mtcars
  lm(formula = mpg ~ cyl + wt)
  ~ lmodel
  summary
  ? .$r.squared
  coef
})
```

ありがとう [@hoxo_m](https://twitter.com/hoxo_m) `pipeline()` のアイデアは[この投稿](http://qiita.com/hoxo_m/items/3fd3d2520fa014a248cb)で提示されました。

## ライセンス

このパッケージは [MIT License](http://opensource.org/licenses/MIT) の下で公開されています。
