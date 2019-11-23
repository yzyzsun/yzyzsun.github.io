---
title: "Megaparsec: Haskell 的语法分析组合子"
author: Mark Karpov
tags: 译文
---

> 原文标题：Megaparsec tutorial from IH book  
> 原文链接：<https://markkarpov.com/megaparsec/megaparsec.html>

<!--more-->

本篇 Megaparsec 教程原本是为《[中级 Haskell](https://intermediatehaskell.com/)》一书写的一章。但由于这本书在过去的一年里没有什么进展，于是其他合著者同意将本文发表为一篇独立的教程，以飨读者。

- 目录
{:toc}

在上一章「例：编写自己的语法分析组合子」中编写的玩具性质的语法分析组合子并不适合实际使用，因此我们继续来看看 Haskell 生态圈中能够解决相同问题的库，并请留意它们各自的利弊权衡：

- [parsec](https://hackage.haskell.org/package/parsec) 过去一直是 Haskell 的「默认」语法分析库。该库比较关注错误信息的质量，但其测试覆盖率不高，并且目前处于仅维护的状态。
- [attoparsec](https://hackage.haskell.org/package/attoparsec) 是个健壮而高性能的语法分析库。它是在列的库中唯一一个完整支持增量语法分析的，其缺点是错误信息质量不佳、用不了单子变换子、只支持部分输入流类型。
- [trifecta](https://hackage.haskell.org/package/trifecta) 错误信息的质量不错，但文档不足导致难以理解。对 `String` 和 `ByteString` 的语法分析可以做到开箱即用，但 `Text` 则不行。
- [megaparsec](https://hackage.haskell.org/package/megaparsec) 是 `parsec` 的一个分支，在过去数年里保持着积极的开发。当前版本尝试在速度、灵活性和错误信息质量之间找到一个最佳平衡。因为是 `parsec` 的非官方继任者，使用过 `parsec` 或者读过其教程的用户一定会对它感到十分亲切。

把上述语法解析库全部讲一遍是不现实的，因此本文聚焦 `megaparsec`。更准确地说，我们将会讲解该库的版本 8.0，其于本书正式发行时应该已经取代旧版本成为主流版本了。

## `ParsecT` 和 `Parsec` 单子

`ParsecT` 是 `megaparsec` 中主要的语法分析单子变换子和核心数据类型。`ParsecT e s m a` 各参数分别表示：

- `e` 是用来表示错误信息的自定义组件的类型。如果我们不想做自定义（目前我们确实不想），那么用 `Data.Void` 模块中的 `Void` 就行了。
- `s` 是输入流的类型。`megaparsec` 对于 `String`、严格或惰性的 `Text`、严格或惰性的 `ByteString` 都是开箱即用的，当然自定义输入流也是可用的。
- `m` 是 `ParsecT` 单子变换子的内部单子。
- `a` 是单子中的值，作为语法分析的结果。

因为大多数时候 `m` 就是 `Identity`，所以 `Parsec` 这个类型别名非常有用：

```haskell
type Parsec e s a = ParsecT e s Identity a
```

简而言之，`Parsec` 就是没有单子变换子的 `ParsecT`。

我们还可以把 `megaparsec` 的单子变换子类比于 MTL 单子变换子和类型类。确实，我们还有 `MonadParsec` 类型类的用途与 `MonadState` 和 `MonadReader` 相近。我们会在[后面的章节](#monadparsec-类型类)详细讨论 `MonadParsec`。

说到类型别名，开始使用 `megaparsec` 的最佳方式就是为自己的语法分析器定义一个类型别名。这有两个好处：

- 添加顶级签名会更容易，例如 `Parser Int`，其中 `Parser` 是你的语法分析单子。没有签名，诸如 `e` 之类的参数会有歧义，这是多态 API 不利的一面。
- 使用固定所有类型变量的具体类型可以帮助 GHC 更好地进行优化，如果你的语法分析器保持多态则 GHC 难以开展优化工作。尽管 `megaparsec` API 是多态的，但预计最终用户都会使用具体类型的语法分析单子，这样便可进行内联工作，并将大多数函数定义转储到接口文件，这让 GHC 能够生成非常高效的非多态代码。

让我们定义一个类型别名（一般都叫 `Parser`）：

```haskell
type Parser = Parsec Void Text
--                   ^    ^
--                   |    |
-- Custom error component Type of input stream
```

在本文中出现的 `Parser` 假定为此类型，直到我们开始自定义语法分析错误为止。

## 字符和二进制流

我们之前说了 `megaparsec` 对于五种输入流类型是开箱即用的：`String`、严格或惰性的 `Text`、严格或惰性的 `ByteString`。之所以可以这样，是因为在该库中这些类型都是 `Stream` 类型类的实例，其对所有可用作 `megaparsec` 语法分析器输入的数据类型做了功能上的抽象。

简化版本的 `Stream` 可以表示如下：

```haskell
class Stream s where
  type Token  s :: *
  type Tokens s :: *
  take1_ :: s -> Maybe (Token s, s) -- aka uncons
  tokensToChunk :: Proxy s -> [Token s] -> Tokens s
```

实际的 `Stream` 定义包含更多方法，但我们不用知道那些就能使用该库。

注意这个类型类关联了两个类型函数：

- `Token s` 是单个词法单词的类型。一般来说是 `Char` 或者 `Word8`，但对于自定义流来说也可能是其他类型。
- `Tokens s` 是流的「一大块」的类型，此概念是为性能考虑而引入的。确实常常有相当于单词列表 `[Token s]` 但又更加高效的表示方法，比如 `Text` 类型的输入流有 `Tokens s ~ Text`，即 `Text` 的一大块就是 `Text`。虽然类型等式 `Tokens s ~ s` 常常是成立的，但在自定义流中 `Tokens s` 和 `s` 可能不同，所以我们将这两个类型分开了。

我们可以把所有默认的输入流列进一张表格：

| `s`                 | `Token s` | `Tokens s`          |
| ------------------- | --------- | ------------------- |
| `String`            | `Char`    | `String`            |
| strict `Text`       | `Char`    | strict `Text`       |
| lazy `Text`         | `Char`    | lazy `Text`         |
| strict `ByteString` | `Word8`   | strict `ByteString` |
| lazy `ByteString`   | `Word8`   | lazy `ByteString`   |

我们得习惯 `Token` 和 `Tokens` 这两个类型函数，因为它们在 `megaparsec` API 的类型声明中无处不在。

你可能会注意到，如果我们把所有默认输入流按照单词类型分类，可以得到两类：

- 字符流，满足 `Token s ~ Char`：`String`、严格或惰性的 `Text`；
- 二进制流，满足 `Token s ~ Word8`：严格或惰性的 `ByteString`。

因此用 `megaparsec` 就不需要为每种输入流类型都编写一个同样的语法分析器（比如用 `attoparsec` 就需要），但我们仍要为不同的单词类型编写不同的代码：

- 使用字符流的组合子，要导入 `Text.Megaparsec.Char` 模块；
- 使用二进制流的组合子，要导入 `Text.Megaparsec.Byte` 模块。

这些模块包含两组相似的语法分析工具，例如：

|           | `Text.Megaparsec.Char`                                | `Text.Megaparsec.Byte`                                 |
| --------- | ----------------------------------------------------- | ------------------------------------------------------ |
| `newline` | `(MonadParsec e s m, Token s ~ Char) => m (Token s)`  | `(MonadParsec e s m, Token s ~ Word8) => m (Token s)`  |
| `eol`     | `(MonadParsec e s m, Token s ~ Char) => m (Tokens s)` | `(MonadParsec e s m, Token s ~ Word8) => m (Tokens s)` |

为了更好地理解我们将使用的工具函数，我们先引入几个它们所依赖的原语。

第一个原语是 `token`，相应地它让我们能够对 `Token s` 类型的值做语法分析：

```haskell
token :: MonadParsec e s m
  => (Token s -> Maybe a)
    -- ^ Matching function for the token to parse
  -> Set (ErrorItem (Token s))
    -- ^ Expected items (in case of an error)
  -> m a
```

`token` 的第一个参数是要分析的单词的匹配函数，如果该函数返回了 `Just` 那么其值就会成为语法分析的结果，`Nothing` 则表明语法分析器不接受该单词并且原语会失败。

第二个参数是一个 `Set`（来自 `containers` 包），它包含在失败的情况下所有可能显示给用户的 `ErrorItem`。当我们讨论语法分析错误时会详细解说 `ErrorItem`。

为了更好地理解 `token` 是怎样工作的，让我们看看 `Text.Megaparsec` 模块中适用于所有输入流类型的一些组合子的定义。`satisfy` 是其中一个相当常见的组合子，我们给它一个对匹配单词返回 `True` 的断言，它就会返回一个对应的语法分析器：

```haskell
satisfy :: MonadParsec e s m
  => (Token s -> Bool) -- ^ Predicate to apply
  -> m (Token s)
satisfy f = token testToken Set.empty
  where
    testToken x = if f x then Just x else Nothing
```

`testToken` 的工作就是把返回 `Bool` 值的 `f` 函数转换为 `token` 期待的返回 `Maybe (Token s)` 的函数。在 `satisfy` 中我们不知道想要匹配的确切单词序列，所以我们传了 `Set.empty` 作为第二个参数。

`satisfy` 看起来很好懂，让我们看看怎么使用它。我们需要一个能跑语法分析器的工具函数，`megaparsec` 提供了 `parseTest` 让我们在 GHCi 中测试。

首先，让我们启动 GHCi 并导入一些模块：

```
λ> import Text.Megaparsec
λ> import Text.Megaparsec.Char
λ> import Data.Text (Text)
λ> import Data.Void
```

我们接着添加 `Parser` 类型别名，以明确语法分析器的类型：

```
λ> type Parser = Parsec Void Text
```

我们还需要开启 `OverloadedStrings` 语言扩展，这样我们就能把字符串字面量用作 `Text` 类型的值：

```
λ> :set -XOverloadedStrings
```
```
λ> parseTest (satisfy (== 'a') :: Parser Char) ""
1:1:
  |
1 | <empty line>
  | ^
unexpected end of input
```
```
λ> parseTest (satisfy (== 'a') :: Parser Char) "a"
'a'
```
```
λ> parseTest (satisfy (== 'a') :: Parser Char) "b"
1:1:
  |
1 | b
  | ^
unexpected 'b'
```
```
λ> parseTest (satisfy (> 'c') :: Parser Char) "a"
1:1:
  |
1 | a
  | ^
unexpected 'a'
```
```
λ> parseTest (satisfy (> 'c') :: Parser Char) "d"
'd'
```

因为 `satisfy` 本身是多态的，所以 `:: Parser Char` 类型标注是必要的，否则 `parseTest` 无法得知 `MonadParsec e s m` 中的 `e` 和 `s` 是什么（在这里 `m` 假定为 `Identity`）。如果我们使用的是一个事先存在的有类型签名的语法分析器，那么就不需要这些显式的类型标注了。

看起来是正常工作的。`satisfy` 有个问题是它没有在失败时告诉我们它期待什么单词，因为我们无法分析 `satisfy` 调用者提供的函数。另外还有一些不那么通用的组合子，但它们生成更有用的错误信息。例如 `single`（还有在 `Text.Megaparsec.Byte` 和 `Texy.Megaparsec.Char` 中限定了类型的别名 `char`）可以匹配一个特定的单词：

```haskell
single :: MonadParsec e s m
  => Token s
  -> m (Token s)
single t = token testToken expected
  where
    testToken x = if x == t then Just x else Nothing
    expected    = E.singleton (Tokens (t:|[]))
```

这里的 `Tokens` 数据类型构造器跟我们之前讨论的 `Tokens` 类型函数没有关系。实际上，`Tokens` 是 `ErrorItem` 的一个构造器，用来指定我们期望匹配的具体单词序列。

```
λ> parseTest (char 'a' :: Parser Char) "b"
1:1:
  |
1 | b
  | ^
unexpected 'b'
expecting 'a'
```
```
λ> parseTest (char 'a' :: Parser Char) "a"
'a'
```

我们现在可以定义之前表格中的 `newline` 了：

```haskell
newline :: (MonadParsec e s m, Token s ~ Char) => m (Token s)
newline = single '\n'
```

第二个原语叫做 `tokens`，它让我们能够语法分析 `Tokens s`，即用来匹配输入的一大块：

```haskell
tokens :: MonadParsec e s m
  => (Tokens s -> Tokens s -> Bool)
    -- ^ Predicate to check equality of chunks
  -> Tokens s
    -- ^ Chunk of input to match against
  -> m (Tokens s)
```

也有两个语法分析器是基于 `tokens` 定义的：

```haskell
-- from "Text.Megaparsec":
chunk :: MonadParsec e s m
  => Tokens s
  -> m (Tokens s)
chunk = tokens (==)
```
```haskell
-- from "Text.Megaparsec.Char" and "Text.Megaparsec.Byte":
string' :: (MonadParsec e s m, CI.FoldCase (Tokens s))
  => Tokens s
  -> m (Tokens s)
string' = tokens ((==) `on` CI.mk)
```

它们会匹配输入中固定的一大块，`chunk`（在 `Text.Megaparsec.Byte` 和 `Texy.Megaparsec.Char` 中有限定了类型的别名 `string`）区分大小写，而 `string'` 不区分。不区分大小写的匹配要用到 `case-insensitive` 包，并且加上了 `FoldCase` 约束。

让我们也来试试这些新的组合子：

```
λ> parseTest (string "foo" :: Parser Text) "foo"
"foo"
```
```
λ> parseTest (string "foo" :: Parser Text) "bar"
1:1:
  |
1 | bar
  | ^
unexpected "bar"
expecting "foo"
```
```
λ> parseTest (string' "foo" :: Parser Text) "FOO"
"FOO"
```
```
λ> parseTest (string' "foo" :: Parser Text) "FoO"
"FoO"
```
```
λ> parseTest (string' "foo" :: Parser Text) "FoZ"
1:1:
  |
1 | FoZ
  | ^
unexpected "FoZ"
expecting "foo"
```

好的，我们可以匹配单个单词和一大块输入了。下一步我们将要学习如何组合这些积木来编写更有趣的语法分析器。

## 单子式和应用函子式语法

最简单的组合语法分析器的方式是连续执行它们。`ParsecT` 和 `Parsec` 是单子，而单子绑定正好可以顺序执行语法分析器：

```haskell
mySequence :: Parser (Char, Char, Char)
mySequence = do
  a <- char 'a'
  b <- char 'b'
  c <- char 'c'
  return (a, b, c)
```

我们来运行一下看看是不是按照预期工作：

```
λ> parseTest mySequence "abc"
('a','b','c')
```
```
λ> parseTest mySequence "bcd"
1:1:
  |
1 | bcd
  | ^
unexpected 'b'
expecting 'a'
```
```
λ> parseTest mySequence "adc"
1:2:
  |
1 | adc
  |  ^
unexpected 'd'
expecting 'b'
```

因为所有单子亦是应用函子，所以我们也可以使用应用函子式的语法来顺序执行：

```haskell
mySequence :: Parser (Char, Char, Char)
mySequence =
  (,,) <$> char 'a'
       <*> char 'b'
       <*> char 'c'
```

第二种方法跟第一种运行结果完全相同，使用哪种风格通常取决于个人品味。单子风格可以说是更冗长但有时更清晰，而应用函子风格通常更简洁。话说回来，显然单子风格的表达能力更强，因为单子比应用函子更强大。

## 用 `eof` 耗尽输入

应用函子通常已经足够强大，足以做一些有趣的事情。如果配上拥有单位元且满足结合律的运算符，我们就得到了应用函子上的幺半群，在 Haskell 中表示为 `Alternative` 类型类。[parser-combinators](https://hackage.haskell.org/package/parser-combinators) 包提供了不少基于 `Applicative` 和 `Alternative` 概念的抽象组合子，`Text.Megaparsec` 模块重新导出了这些来自 `Control.Applicative.Combinators` 的组合子。

一个最常见的组合子是 `many`，它允许我们将给定的语法分析器运行零次或多次：

```
λ> parseTest (many (char 'a') :: Parser [Char]) "aaa"
"aaa"
```
```
λ> parseTest (many (char 'a') :: Parser [Char]) "aabbb"
"aa"
```

第二个结果可能有点令人惊讶，语法分析器吃掉了匹配的 `a`，但随后就停了下来。好吧，我们并没有交代在 `many (char 'a')` 之后要做些什么。

在大多数情况下，我们实际需要强制语法分析器吃掉整个输入，并报告语法分析错误，而不是害羞地默默中止。这就需要我们吃到输入结束。幸运的是，虽然输入结束只是一个概念，但有个 `eof :: MonadParsec e s m => m ()` 不吃任何单词，仅会在输入结束时成功。让我们把它加进去再试一次：

```
λ> parseTest (many (char 'a') <* eof :: Parser [Char]) "aabbb"
1:3:
  |
1 | aabbb
  |   ^
unexpected 'b'
expecting 'a' or end of input
```

我们在语法分析器中没有提到 `b`，所以它们肯定是预期之外的。

## 处理多种选择

从现在开始我们将开发一个实际有用的语法分析器，它能处理下述形式的 URI：

```
scheme:[//[user:password@]host[:port]][/]path[?query][#fragment]
```

我们记住方括号 `[]` 中的部分是可选的，不论它们出不出现 URI 都是合法的，`[]` 甚至可以进行嵌套。我们会完整支持该语法[^modern-uri]。

[^modern-uri]: 实际上有个 [modern-uri](https://hackage.haskell.org/package/modern-uri) 包，其 Megaparsec 语法分析器支持 RFC 3986 定义的 URI 格式，但它远比我们这里介绍的要复杂。

让我们从 `scheme` 开始，我们仅仅接受已知的协议名，譬如 `data`、`file`、`ftp`、`http`、`https`、`irc` 和 `mailto`。

我们用 `string` 匹配固定的字符序列，用 `Alternative` 类型类中的 `(<|>)` 方法表示「选择」。代码如下：

```haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE RecordWildCards   #-}

module Main (main) where

import Control.Applicative
import Control.Monad
import Data.Text (Text)
import Data.Void
import Text.Megaparsec hiding (State)
import Text.Megaparsec.Char
import qualified Data.Text as T
import qualified Text.Megaparsec.Char.Lexer as L

type Parser = Parsec Void Text

pScheme :: Parser Text
pScheme = string "data"
  <|> string "file"
  <|> string "ftp"
  <|> string "http"
  <|> string "https"
  <|> string "irc"
  <|> string "mailto"
```

试着运行一下：

```
λ> parseTest pScheme ""
1:1:
  |
1 | <empty line>
  | ^
unexpected end of input
expecting "data", "file", "ftp", "http", "https", "irc", or "mailto"
```
```
λ> parseTest pScheme "dat"
1:1:
  |
1 | dat
  | ^
unexpected "dat"
expecting "data", "file", "ftp", "http", "https", "irc", or "mailto"
```
```
λ> parseTest pScheme "file"
"file"
```
```
λ> parseTest pScheme "irc"
"irc"
```

看起来不错，但 `pScheme` 的定义有点啰嗦。我们可以用 `choice` 组合子重写 `pScheme`：

```haskell
pScheme :: Parser Text
pScheme = choice
  [ string "data"
  , string "file"
  , string "ftp"
  , string "http"
  , string "https"
  , string "irc"
  , string "mailto" ]
```

`choice` 只是 `asum` 的别名，后者会用 `(<|>)` 对列表元素进行折叠，所以 `pScheme` 的这两个定义其实是一样的，只是用 `choice` 更好看一些。

协议名后面是个冒号 `:`。回忆一下，如果我们要接着对其他东西做语法分析，我们要用单子绑定或者 `do` 记法：

```haskell
data Uri = Uri
  { uriScheme :: Text
  } deriving (Eq, Show)

pUri :: Parser Uri
pUri = do
  r <- pScheme
  _ <- char ':'
  return (Uri r)
```

如果我们运行一下 `pUri`，我们会看到它现在要求协议名后面跟着一个冒号：

```
λ> parseTest pUri "irc"
1:4:
  |
1 | irc
  |    ^
unexpected end of input
expecting ':'
```
```
λ> parseTest pUri "irc:"
Uri {uriScheme = "irc"}
```

但我们还没完成协议名的语法分析。一位优秀的 Haskell 程序员编写的类型，能让错误数据无处遁形。并不是任何 `Text` 值都代表合法的协议名，因此让我们定义一个表示协议名的数据类型，并让 `pScheme` 返回它：

```haskell
data Scheme
  = SchemeData
  | SchemeFile
  | SchemeFtp
  | SchemeHttp
  | SchemeHttps
  | SchemeIrc
  | SchemeMailto
  deriving (Eq, Show)

pScheme :: Parser Scheme
pScheme = choice
  [ SchemeData   <$ string "data"
  , SchemeFile   <$ string "file"
  , SchemeFtp    <$ string "ftp"
  , SchemeHttp   <$ string "http"
  , SchemeHttps  <$ string "https"
  , SchemeIrc    <$ string "irc"
  , SchemeMailto <$ string "mailto" ]

data Uri = Uri
  { uriScheme :: Scheme
  } deriving (Eq, Show)
```

`(<$)` 运算符仅仅把左边的值放入函子上下文，而不管里面原来是什么。`a <$ f` 与 `const a <$> f` 等价，但对于一些函子来说更高效。

让我们再来试一下：

```
λ> parseTest pUri "https:"
1:5:
  |
1 | https:
  |     ^
unexpected 's'
expecting ':'
```

唔……`https` 应该是个合法的协议名，你能看出哪里出错了吗？语法分析器逐个尝试这些选择，一旦 `http` 匹配就不会再往下试 `https` 了。解决方法就是把 `SchemeHttps <$ string "https"` 放到 `schemeHttp <$ string "http"` 上面去。一定要记住：顺序会影响选择！

现在 `pUri` 正常工作了：

```
λ> parseTest pUri "http:"
Uri {uriScheme = SchemeHttp}
```
```
λ> parseTest pUri "https:"
Uri {uriScheme = SchemeHttps}
```
```
λ> parseTest pUri "mailto:"
Uri {uriScheme = SchemeMailto}
```
```
λ> parseTest pUri "foo:"
1:1:
  |
1 | foo:
  | ^
unexpected "foo:"
expecting "data", "file", "ftp", "http", "https", "irc", or "mailto"
```

## 用 `try` 控制回溯

下一步是处理 `[//[user:password@]host[:port]]`，这里我们需要嵌套可选部分，因此让我们更新一下 `Uri` 类型：

```haskell
data Uri = Uri
  { uriScheme    :: Scheme
  , uriAuthority :: Maybe Authority
  } deriving (Eq, Show)

data Authority = Authority
  { authUser :: Maybe (Text, Text) -- (user, password)
  , authHost :: Text
  , authPort :: Maybe Int
  } deriving (Eq, Show)
```

现在我们需要讨论一个重要概念，也就是回溯。回溯是指及时返回而不吃掉任何输入，这在处理分支是非常重要。下面是一个例子：

```haskell
alternatives :: Parser (Char, Char)
alternatives = foo <|> bar
  where
    foo = (,) <$> char 'a' <*> char 'b'
    bar = (,) <$> char 'a' <*> char 'c'
```

看起来很合理，我们来试一下：

```
λ> parseTest alternatives "ab"
('a','b')
```
```
λ> parseTest alternatives "ac"
1:2:
  |
1 | ac
  |  ^
unexpected 'c'
expecting 'b'
```

发生了什么呢？最先尝试的 `foo` 的 `char 'a'` 部分成功了，所以输入流里的 `a` 被吃掉了。接着 `char 'b'` 没能匹配 `c`，所以我们得到了这样的错误信息。重要的一点是，`(<|>)` 根本没有尝试 `bar`，因为 `foo` 已经把一些输入吃掉了。

一方面这是为性能考虑，另一方面把 `foo` 剩下来的东西喂给 `bar` 也没什么意义。我们期望在 `bar` 运行的时候，输入流正处于 `foo` 开始的位置。`megaparsec` 并不会自动回溯（与 `attoparsec` 或是上一章中的玩具组合子不同），所以我们需要用 `try` 原语来显式表达我们想要回溯。如果 `p` 失败了，那么 `try p` 就会进行回溯，就像没有输入被吃掉一样（实际上它回溯了整个语法分析状态）。这就允许 `(<|>)` 尝试右边的选择了：

```haskell
alternatives :: Parser (Char, Char)
alternatives = try foo <|> bar
  where
    foo = (,) <$> char 'a' <*> char 'b'
    bar = (,) <$> char 'a' <*> char 'c'
```
```
λ> parseTest alternatives "ac"
('a','c')
```

所有会吃输入的原语（当然也有诸如 `try` 这样改变现有语法分析器行为的原语）的输入消耗是「原子性」的。也就是说，它们失败时会自动回溯，所以它们不会吃掉部分输入而中途失败。这就是为什么 `pScheme` 的所有选择能正常工作：`string` 是基于 `tokens` 定义的，而 `tokens` 是原语。我们要么匹配整个字符串，要么直接失败而不吃掉任何输入流。

回到 URI 的语法分析上，`(<|>)` 能够用来构建一个方便的 `optional` 组合子：

```haskell
optional :: Alternative f => f a -> f (Maybe a)
optional p = (Just <$> p) <|> pure Nothing
```

如果 `optional p` 中的 `p` 匹配成功了，那么我们能得到包装在 `Just` 中的结果，否则返回 `Nothing`。这就是我们想要的！但我们没必要自己定义 `optional`，因为 `Text.Megaparsec` 帮我们重新导出了这个组合子。我们现在可以把它用在 `pUri` 上了：

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- pScheme
  void (char ':')
  uriAuthority <- optional . try $ do            -- (1)
    void (string "//")
    authUser <- optional . try $ do              -- (2)
      user <- T.pack <$> some alphaNumChar       -- (3)
      void (char ':')
      password <- T.pack <$> some alphaNumChar
      void (char '@')
      return (user, password)
    authHost <- T.pack <$> some (alphaNumChar <|> char '.')
    authPort <- optional (char ':' *> L.decimal) -- (4)
    return Authority {..}                        -- (5)
  return Uri {..}                                -- (6)
```

我擅自让所有字母和数字都可用作用户名和密码，主机名也做了相似的简化。

有几点需要留意：

- 在 (1) 和 (2) 中我们需要把 `optional` 的参数用 `try` 包起来，因为其参数是组合起来的语法分析器，而不是原语。
- (3) `some` 和 `many` 很像，但要求至少匹配一次：`some p = (:) <$> p <*> many p`。
- (4) 如非必要请勿使用 `try`！这里如果 `char ':'` 成功了（它本身是基于 `token` 定义的，不需要 `try`），我们知道紧接着一定是端口号，所以我们只需要 `L.decimal` 来匹配十进制数。在匹配完 `:` 之后，我们并不需要回溯。
- 在 (5) 和 (6) 中我们用 `RecordWildCards` 语言扩展组装了 `Authority` 和 `Uri` 的值。

在 GHCi 中试试 `pUri`，你会发现它能正常工作：

```
λ> parseTest (pUri <* eof) "https://mark:secret@example.com"
Uri
  { uriScheme = SchemeHttps
  , uriAuthority = Just (Authority
    { authUser = Just ("mark","secret")
    , authHost = "example.com"
    , authPort = Nothing } ) }
```
```
λ> parseTest (pUri <* eof) "https://mark:secret@example.com:123"
Uri
  { uriScheme = SchemeHttps
  , uriAuthority = Just (Authority
    { authUser = Just ("mark","secret")
    , authHost = "example.com"
    , authPort = Just 123 } ) }
```
```
λ> parseTest (pUri <* eof) "https://example.com:123"
Uri
  { uriScheme = SchemeHttps
  , uriAuthority = Just (Authority
    { authUser = Nothing
    , authHost = "example.com"
    , authPort = Just 123 } ) }
```
```
λ> parseTest (pUri <* eof) "https://mark@example.com:123"
1:13:
  |
1 | https://mark@example.com:123
  |             ^
unexpected '@'
expecting '.', ':', alphanumeric character, or end of input
```

## 调试语法分析器

不过，你可能会发现这样一个问题：

```
λ> parseTest (pUri <* eof) "https://mark:@example.com"
1:7:
  |
1 | https://mark:@example.com
  |       ^
unexpected '/'
expecting end of input
```

这个语法分析错误的提示信息有待改进！怎么改进呢？弄清问题所在的最简单方法是用内置的 `dbg` 工具：

```haskell
dbg :: (Stream s, ShowToken (Token s), ShowErrorComponent e, Show a)
  => String            -- ^ Debugging label
  -> ParsecT e s m a   -- ^ Parser to debug
  -> ParsecT e s m a   -- ^ Parser that prints debugging messages
```

让我们把它加进 `pUri`：

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- dbg "scheme" pScheme
  void (char ':')
  uriAuthority <- dbg "auth" . optional . try $ do
    void (string "//")
    authUser <- dbg "user" . optional . try $ do
      user <- T.pack <$> some alphaNumChar
      void (char ':')
      password <- T.pack <$> some alphaNumChar
      void (char '@')
      return (user, password)
    authHost <- T.pack <$> dbg "host" (some (alphaNumChar <|> char '.'))
    authPort <- dbg "port" $ optional (char ':' *> L.decimal)
    return Authority {..}
  return Uri {..}
```

然后再用刚才的输入运行 `pUri` 看看：

```
λ> parseTest (pUri <* eof) "https://mark:@example.com"
scheme> IN: "https://mark:@example.com"
scheme> MATCH (COK): "https"
scheme> VALUE: SchemeHttps

user> IN: "mark:@example.com"
user> MATCH (EOK): <EMPTY>
user> VALUE: Nothing

host> IN: "mark:@example.com"
host> MATCH (COK): "mark"
host> VALUE: "mark"

port> IN: ":@example.com"
port> MATCH (CERR): ':'
port> ERROR:
port> 1:14:
port> unexpected '@'
port> expecting integer

auth> IN: "//mark:@example.com"
auth> MATCH (EOK): <EMPTY>
auth> VALUE: Nothing

1:7:
  |
1 | https://mark:@example.com
  |       ^
unexpected '/'
expecting end of input
```

我们可以看到内部到底发生什么了：

- `scheme` 匹配成功了。
- `user` 失败了：虽然有个用户名 `mark`，但 `:` 后面没有密码（我们这里要求密码非空）。虽然我们失败了，但 `try` 带我们回溯了。
- `host` 从 `user` 开始的位置运作，并尝试把输入解释成主机名。我们可以看到它成功了，并返回了主机名 `mark`。
- 主机名后面可以有端口号，所以 `port` 开始运作。它看见了 `:`，但发现后面没有数字，因此也失败了。
- 因此整个 `auth` 语法分析失败了（`port` 在 `auth` 里面失败了）。
- `auth` 语法分析器返回了 `Nothing`，因为它什么都分析不出来。现在 `eof` 要求吃到输入结束，但现实并非如此，因此我们得到了最终的错误信息。

我们怎么办呢？这是一个用 `try` 包住一大堆代码导致错误信息不可读的例子。让我们再看一下我们要处理的语法：

```
scheme:[//[user:password@]host[:port]][/]path[?query][#fragment]
```

我们在找什么？在找可以让我们确定特定分支的东西，就像我们看到 `:` 就肯定后面跟着端口号一样。如果仔细找的话，你会发现双斜杠是 `//` 是进入 `Authority` 部分的标志。因为我们是用原子性的语法分析器（`string`）来匹配 `//` 的，它会自动回溯，而一旦匹配到 `//` 我们就能确定我们需要匹配到 `Authority` 部分。让我们把 `pUri` 的第一个 `try` 删掉吧：

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- pScheme
  void (char ':')
  uriAuthority <- optional $ do -- removed 'try' on this line
    void (string "//")
    authUser <- optional . try $ do
      user <- T.pack <$> some alphaNumChar
      void (char ':')
      password <- T.pack <$> some alphaNumChar
      void (char '@')
      return (user, password)
    authHost <- T.pack <$> some (alphaNumChar <|> char '.')
    authPort <- optional (char ':' *> L.decimal)
    return Authority {..}
  return Uri {..}
```

现在我们得到了更可读的错误信息：

```
λ> parseTest (pUri <* eof) "https://mark:@example.com"
1:14:
  |
1 | https://mark:@example.com
  |              ^
unexpected '@'
expecting integer
```

虽然有点误导人，但这是个比较微妙的例子。里面有太多 `optional` 了。

## 标签和隐藏

有时完整列出我们期待的东西会有点长，记得我们用未知的协议名进行测试的时候吗？

```
λ> parseTest (pUri <* eof) "foo://example.com"
1:1:
  |
1 | foo://example.com
  | ^
unexpected "foo://"
expecting "data", "file", "ftp", "http", "https", "irc", or "mailto"
```

`megaparsec` 提供了对提示信息进行自定义的方法，即使用「标签」。我们可以这样使用 `label` 原语（它有个别名是 `(<?>)` 运算符）：

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- pScheme <?> "valid scheme"
  -- the rest stays the same
```
```
λ> parseTest (pUri <* eof) "foo://example.com"
1:1:
  |
1 | foo://example.com
  | ^
unexpected "foo://"
expecting valid scheme
```

我们可以继续加入更多标签，以使错误信息更加可读：

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- pScheme <?> "valid scheme"
  void (char ':')
  uriAuthority <- optional $ do
    void (string "//")
    authUser <- optional . try $ do
      user <- T.pack <$> some alphaNumChar <?> "username"
      void (char ':')
      password <- T.pack <$> some alphaNumChar <?> "password"
      void (char '@')
      return (user, password)
    authHost <- T.pack <$> some (alphaNumChar <|> char '.') <?> "hostname"
    authPort <- optional (char ':' *> label "port number" L.decimal)
    return Authority {..}
  return Uri {..}
```

举个例子：

```
λ> parseTest (pUri <* eof) "https://mark:@example.com"
1:14:
  |
1 | https://mark:@example.com
  |              ^
unexpected '@'
expecting port number
```

另一个原语叫做 `hidden`。如果说 `label` 是在为提示信息进行重命名，那么 `hidden` 就是把它们直接移除了。做个比较：

```
λ> parseTest (many (char 'a') >> many (char 'b') >> eof :: Parser ()) "d"
1:1:
  |
1 | d
  | ^
unexpected 'd'
expecting 'a', 'b', or end of input
```
```
λ> parseTest (many (char 'a') >> hidden (many (char 'b')) >> eof :: Parser ()) "d"
1:1:
  |
1 | d
  | ^
unexpected 'd'
expecting 'a' or end of input
```

如果我们想让错误信息不那么啰嗦，`hidden` 会很有用。比如说，在对编程语言做语法分析时，最好丢弃「expecting white space」的提示信息，因为几乎所有单词后面都可以有空格。

【练习】我们把 `pUri` 的剩余部分留给读者完成，所有要用到的工具都已经讲解过了。

## 运行语法分析器

我们已经探索了如何构建语法分析器，但我们还没审视能让我们运行它们的函数，除了 `parseTest`。

传统上来说，从你的程序运行语法分析器的「默认」函数一直是 `parse`，但`parse` 其实是 `runParser` 的别名：

```haskell
runParser
  :: Parsec e s a -- ^ Parser to run
  -> String       -- ^ Name of source file
  -> s            -- ^ Input for parser
  -> Either (ParseErrorBundle s e) a
```

第二个参数只是用来在错误信息中显示的文件名，`megaparsec` 并不会尝试去读这个文件，因为真正的输入是这个函数的第三个参数。

`runParser` 允许我们运行 `Parsec` 单子，我们已经知道，它就是没有单子变换子的 `ParsecT`：

```haskell
type Parsec e s = ParsecT e s Identity
```

`runParser` 有三个姊妹：`runParser'`、`runParserT` 和 `runParserT'`。有后缀 `T` 的版本可以运行 `ParsecT` 单子变换子，而有一撇的版本接受并返回语法分析器状态。让我们把它们列进一张表：

| 参数           | 运行 `Parsec` | 运行 `ParsecT` |
| -------------- | ------------- | -------------- |
| 输入和文件名   | `runParser`   | `runParserT`   |
| 自定义初始状态 | `runParser'`  | `runParserT'`  |

比如当你需要设置制表符宽度（默认是 8）的时候，自定义初始状态就很有用。举了例子，下面是 `runParser'` 的类型签名：

```haskell
runParser'
  :: Parsec e s a -- ^ Parser to run
  -> State s      -- ^ Initial state
  -> (State s, Either (ParseErrorBundle s e) a)
```

手动修改 `State` 是该库的高级用法，我们不会在这里介绍。

## `MonadParsec` 类型类

`megaparsec` 中的所有工具都可用于 `MonadParsec` 类型类的任何实例。该类型类抽象了「组合子原语」，即所有 `megaparsec` 语法分析器的基本单元，这些组合子无法用其它组合子来表示。

将组合子原语定义为类型类，让 `ParsecT` 主要的具体单子变换子得以包装在我们熟悉的 MTL 系变换子中，从而实现在单子栈各层之间的不同交互。为了更好地理解其动机，请回忆一下单子栈各层的顺序很重要。如果我们这样组合 `ReaderT` 和 `State`：

```haskell
type MyStack a = ReaderT MyContext (State MyState) a
```

在外层，`ReaderT` 无法检查里面 `m` 层的内部结构。`ReaderT` 的 `Monad` 实例描述了绑定策略：

```haskell
newtype ReaderT r m a = ReaderT { runReaderT :: r -> m a }

instance Monad m => Monad (ReaderT r m) where
  m >>= k = ReaderT $ \r -> do
    a <- runReaderT m r
    runReaderT (k a) r
```

实际上，我们对 `m` 的唯一了解是它是 `Monad` 的一个实例，因此 `m` 的状态只能通过单子绑定传给 `k`。总之，这就是 `ReaderT` 的 `(>>=)` 起到的典型作用。

`Alternative` 的 `(<|>)` 方法有着不同的作用：它「分裂」了状态，并且这两个语法分析分支不再有交集，因此在某种意义上我们可以回溯状态。也就是说，如果第一个分支被丢弃，那么它对状态的修改也会被丢弃，并不会影响到第二个分支（相当于我们在第一个分支失败时回溯了状态）。

举个例子，我们可以看看 `ReaderT` 的 `Alternative` 实例：

```haskell
instance Alternative m => Alternative (ReaderT r m) where
  empty = liftReaderT empty
  ReaderT m <|> ReaderT n = ReaderT $ \r -> m r <|> n r
```

这很棒，因为 `ReaderT` 是个「无状态」的单子变换子，并且很容易将实际工作委托给内部单子（在这里 `m` 的 `Alternative` 实例很有用），而无需组合 `ReaderT` 自身的单子状态（它并没有）。

现在我们来看看 `State`，因为 `State s a` 只是 `StateT s Identity a` 的别名，我们应该看看 `StateT s m` 的 `Alternative` 实例：

```haskell
instance (Functor m, Alternative m) => Alternative (StateT s m) where
  empty = StateT $ \_ -> empty
  StateT m <|> StateT n = StateT $ \s -> m s <|> n s
```

这里我们看到了状态 `s` 的分裂，正如我们看到了上下文 `r` 的共享。它们的区别是，表达式 `m s` 和 `n s` 会产生有状态的结果：除了单子中的值，它们还会在元组中返回新的状态。这里我们要么走 `m s` 要么走 `n s`，自然实现了回溯。

`ParsecT` 又如何呢？让我们考虑像下面这样把 `State` 放进 `ParsecT`：

```haskell
type MyStack a = Parsec Void Text (State MyState) a
```

`ParsecT` 比 `ReaderT` 更复杂，所以它的 `(<|>)` 实现得做更多工作：

- 管理语法分析器本身的状态；
- 合并语法分析错误，如果有的话。

因此 `ParsecT` 的 `Alternative` 实例中的 `(<|>)` 实现无法将其工作委托给内部单子 `State MyState` 的 `Alternative` 实例，所以 `MyState` 不会分裂，我们也不能回溯。

让我们用一个例子来证明这一点：

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main (main) where

import Control.Applicative
import Control.Monad.State.Strict
import Data.Text (Text)
import Data.Void
import Text.Megaparsec hiding (State)

type Parser = ParsecT Void Text (State String)

parser0 :: Parser String
parser0 = a <|> b
  where
    a = "foo" <$ put "branch A"
    b = get   <* put "branch B"

parser1 :: Parser String
parser1 = a <|> b
  where
    a = "foo" <$ put "branch A" <* empty
    b = get   <* put "branch B"

main :: IO ()
main = do
  let run p          = runState (runParserT p "" "") "initial"
      (Right a0, s0) = run parser0
      (Right a1, s1) = run parser1

  putStrLn  "Parser 0"
  putStrLn ("Result:      " ++ show a0)
  putStrLn ("Final state: " ++ show s0)

  putStrLn  "Parser 1"
  putStrLn ("Result:      " ++ show a1)
  putStrLn ("Final state: " ++ show s1)
```

这是程序的运行结果：

```
Parser 0
Result:      "foo"
Final state: "branch A"
Parser 1
Result:      "branch A"
Final state: "branch B"
```

我们可以看到 `parser0` 的分支 `b` 没有被尝试过。而 `parser1` 的最终结果（`get` 返回的值）显然来自分支 `a`，即使它因为 `empty` 而失败了，成功的是分支 `b`（`empty` 在这里表示立即失败，并且不会提供任何提示信息）。并没有发生回溯。

如果我们想要回溯自定义的状态怎么办呢？如果允许将 `ParsecT` 包装在 `StateT` 里面的话，就可以做到：

```haskell
type MyStack a = StateT MyState (ParsecT Void Text Identity) a
```

现在我们在 `MyStack` 上用的 `(<|>)` 作用于 `StateT` 的实例：

```haskell
StateT m <|> StateT n = StateT $ \s -> m s <|> n s
```

它会帮我们回溯状态，并会把剩下的工作委托给内部单子 `ParsecT` 的 `Alternative` 实例。这样的行为就是我们想要的：

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main (main) where

import Control.Applicative
import Control.Monad.Identity
import Control.Monad.State.Strict
import Data.Text (Text)
import Data.Void
import Text.Megaparsec hiding (State)

type Parser = StateT String (ParsecT Void Text Identity)

parser :: Parser String
parser = a <|> b
  where
    a = "foo" <$ put "branch A" <* empty
    b = get   <* put "branch B"

main :: IO ()
main = do
  let p            = runStateT parser "initial"
      Right (a, s) = runParser p "" ""
  putStrLn ("Result:      " ++ show a)
  putStrLn ("Final state: " ++ show s)
```

程序输出为：

```
Result:      "initial"
Final state: "branch B"
```

为了让这种方法可行，`StateT` 应当支持所有组合子原语，这样我们就能像 `ParsecT` 一样使用它们。换句话说，它们应当是 `MonadParsec` 的实例，就像它们不仅是 `MonadState` 的实例，还是 `MonadWriter` 的实例，只要它们的内部单子也是 `MonadWriter` 的实例：

```haskell
instance MonadWriter w m => MonadWriter w (StateT s m) where …
```

实际上，我们可以将原语从 `MonadParsec` 的内部实例提升到 `StateT`：

```haskell
instance MonadParsec e s m => MonadParsec e s (StateT st m) where …
```

`megaparsec` 为所有 MTL 单子变换子定义了 `MonadParsec` 的实例，这样用户就可以自由地在 `ParsecT` 中插入变换子，或是把 `ParsecT` 包装在那些变换子中，从而实现在单子栈各层之间的不同交互。

## 词法分析

词法分析是将输入流转换为词法单词流的过程：整数、关键字、符号等等，它们比原始输入更加容易直接分析，或者可以送作生成的语法分析器的输入。词法分析可以用外部工具（如 `alex`）单独一个流程去做，但 `megaparsec` 也提供了可以无缝衔接编写词法分析器的函数。

共有两个词法分析器模块：`Text.Megaparsec.Char.Lexer` 用来处理字符流，`Text.Megaparsec.Byte.Lexer` 用来处理字节流。因为我们的输入流是严格求值的 `Text`，所以我们会用 `Text.Megaparsec.Char.Lexer`，不过大多数函数在 `Text.Megaparsec.Byte.Lexer` 里面长得差不多。

### 空格

我们要讨论的第一个话题是如何处理空格。在消耗空格的时候保持一致性比较好，即要么在单词前要么在后。`megaparsec` 的词法分析器模块遵循的策略是：假设单词前没有空格，消耗单词后的空格。

我们需要一种特殊的词法分析器来消耗空格，我们叫它空格消耗器。`Text.Megaparsec.Char.Lexer` 模块提供了构建通用的空格消耗器的工具：

```haskell
space :: MonadParsec e s m
  => m () -- ^ A parser for space characters which does not accept empty input (e.g. 'space1')
  -> m () -- ^ A parser for a line comment (e.g. 'skipLineComment')
  -> m () -- ^ A parser for a block comment (e.g. 'skipBlockComment')
  -> m ()
```

`space` 函数的文档挺好理解的，但还是让我们来举例说明吧：

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main (main) where

import Data.Text (Text)
import Data.Void
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Text.Megaparsec.Char.Lexer as L -- (1)

type Parser = Parsec Void Text

sc :: Parser ()
sc = L.space
  space1                         -- (2)
  (L.skipLineComment "//")       -- (3)
  (L.skipBlockComment "/*" "*/") -- (4)
```

- (1) `Text.Megaparsec.Char.Lexer` 应当限定导入，因为它包含会与 `Text.Megaparsec.Char` 等模块冲突的名字，比如 `space`。
- (2) `L.space` 的第一个参数是个挑选空格的词法分析器。要注意它不能接受空输入，否则 `L.space` 会陷入死循环。`Text.Megaparsec.Char` 里的 `space1` 完美符合要求。
- (3) `L.space` 的第二个参数定义了如何跳过行注释，即以给定单词序列开始、以行末结束的注释。`skipLineComment` 可以帮我们轻松创建一个这样的词法分析器。
- (4) `L.space` 的第三个参数定义了如何跳过块注释，即包裹在给定的始末单词序列中的注释。`skipBlockComment` 可以帮我们处理非嵌套的块注释，若要支持嵌套则可使用 `skipBlockCommentNested`。

操作上，`L.space` 会不停地轮流尝试以上三个词法分析器，直到三个都不再消耗空格。如果你的语法不包含注释，那么可以直接把 `empty` 作为第二或第三个参数送给 `L.space`。`empty`，作为 `(<|>)` 的单位元，仅仅会让 `L.space` 尝试下一个词法分析器。

有了空格消耗器 `sc`，我们可以定义各种空格相关的工具：

```haskell
lexeme :: Parser a -> Parser a
lexeme = L.lexeme sc

symbol :: Text -> Parser Text
symbol = L.symbol sc
```

- `lexeme` 是对词汇分析器的一种包装，能用已给定的空格消耗器挑选出所有尾随空格；
- `symbol` 在内部使用 `string` 来匹配文本，类似地能够挑选出所有的尾随空格。

稍后我们将看到它们如何协同工作，但在此之前我们需要引入更多来自 `Text.Megaparsec.Char.Lexer` 的工具。

### 字符和字符串字面量

对字符和字符串字面量进行词法分析比较微妙，因为有太多转义规则。简单起见，`megaparsec` 提供了 `charLiteral` 词法分析器：

```haskell
charLiteral :: (MonadParsec e s m, Token s ~ Char) => m Char
```

`charLiteral` 的工作是根据 Haskell 报告中描述的字符字面量语法来对可能转义了的单个字符进行词法分析。但注意它不会管字面量两边的引号，这有两个原因：

- 用户可以控制字符字面量用什么作为引号；
- `charLiteral` 也可以用来对字符串字面量进行词法分析。

下面是基于 `charLiteral` 构建词法分析器的例子：

```haskell
charLiteral :: Parser Char
charLiteral = between (char '\'') (char '\'') L.charLiteral

stringLiteral :: Parser String
stringLiteral = char '\"' *> manyTill L.charLiteral (char '\"')
```

- 要把 `L.charLiteral` 改造成我们所需的字符字面量的词法分析器，只需要加上两边的引号。这里我们遵循 Haskell 语法用了单引号。`between` 组合子是这样定义的：`between open close p = open *> p <* close`。
- `stringLiteral` 用 `L.charLiteral` 来对每个字符进行词法分析，两边则用双引号包裹。

第二个函数也很有趣，因为它用了 `manyTill` 组合子：

```haskell
manyTill :: Alternative m => m a -> m end -> m [a]
manyTill p end = go
  where
    go = ([] <$ end) <|> ((:) <$> p <*> go)
```

每一轮 `manyTill` 先尝试运行 `end` 词法分析器，如果失败了就运行 `p` 并把结果装进列表。也有 `someTill` 保证 `p` 至少成功一次。

### 数字

最后，一个非常常见的需求是对数字进行词法分析。对于整数来说，有三种工具分别处理十进制、八进制和十六进制数：

```haskell
decimal, octal, hexadecimal
  :: (MonadParsec e s m, Token s ~ Char, Num a) => m a
```

使用起来很简单：

```haskell
integer :: Parser Integer
integer = lexeme L.decimal
```
```
λ> parseTest (integer <* eof) "123  "
123
```
```
λ> parseTest (integer <* eof) "12a  "
1:3:
  |
1 | 12a
  |   ^
unexpected 'a'
expecting end of input or the rest of integer
```

`scientific` 接受整数和小数的语法，而 `float` 只接受小数。`scientific` 会返回 `scientific` 包的 `Scientific` 类型，而 `float` 的返回类型是多态的，可能会返回任何 `RealFloat` 的实例：

```haskell
scientific :: (MonadParsec e s m, Token s ~ Char)              => m Scientific
float      :: (MonadParsec e s m, Token s ~ Char, RealFloat a) => m a
```

举个例子：

```haskell
float :: Parser Double
float = lexeme L.float
```
```
λ> parseTest (float <* eof) "123"
1:4:
  |
1 | 123
  |    ^
unexpected end of input
expecting '.', 'E', 'e', or digit
```
```
λ> parseTest (float <* eof) "123.45"
123.45
```
```
λ> parseTest (float <* eof) "123d"
1:4:
  |
1 | 123d
  |    ^
unexpected 'd'
expecting '.', 'E', 'e', or digit
```

注意所有这些词法分析器都无法处理有符号数，要支持这个我们得把它们包装在 `signed` 组合子中：

```haskell
signedInteger :: Parser Integer
signedInteger = L.signed sc integer

signedFloat :: Parser Double
signedFloat = L.signed sc float
```

`signed` 的第一个参数是空格消耗器，用来控制正负号和实际数字之间的空格。如果你不允许中间有空格，传 `return ()` 进去就行了。

## `notFollowedBy` 和 `lookAhead`

除了 `try`，还有另外两种原语可以对输入流进行前瞻，而不会实际挪动当前位置。

第一种是 `notFollowedBy`：

```haskell
notFollowedBy :: MonadParsec e s m => m a -> m ()
```

只有当其参数语法分析失败了它才会成功，并且不会吃掉任何输入或是修改当前状态。

作为 `notFollowedBy` 的例子，我们考虑一下关键字：

```haskell
pKeyword :: Text -> Parser Text
pKeyword keyword = lexeme (string keyword)
```

这个语法分析器有个毛病：如果我们匹配到的只是标识符的前缀怎么办呢？这个情况下它显然不是关键字。因此我们必须用 `notFollowedBy` 排除这种情况：

```haskell
pKeyword :: Text -> Parser Text
pKeyword keyword = lexeme (string keyword <* notFollowedBy alphaNumChar)
```

另一种原语是 `lookAhead`：

```haskell
lookAhead :: MonadParsec e s m => m a -> m a
```

如果 `lookAhead` 的参数 `p` 成功了，那么整个 `lookAhead p` 也会成功，但输入流和整个语法分析状态不会改变。

一个例子是对已分析的输入进行检查，要么失败要么成功地进行下去。这可以用下述代码表达：

```haskell
withPredicate1
  :: (a -> Bool)       -- ^ The check to perform on parsed input
  -> String            -- ^ Message to print when the check fails
  -> Parser a          -- ^ Parser to run
  -> Parser a          -- ^ Resulting parser that performs the check
withPredicate1 f msg p = do
  r <- lookAhead p
  if f r
    then p
    else fail msg
```

这演示了 `lookAhead` 的一种用法，但我们还应注意，如果检查成功我们会进行两次语法分析，这不太好。我们可以改用 `getOffset` 函数解决这个问题：

```haskell
withPredicate2
  :: (a -> Bool)       -- ^ The check to perform on parsed input
  -> String            -- ^ Message to print when the check fails
  -> Parser a          -- ^ Parser to run
  -> Parser a          -- ^ Resulting parser that performs the check
withPredicate2 f msg p = do
  o <- getOffset
  r <- p
  if f r
    then return r
    else do
      setOffset o
      fail msg
```

在失败时，我们只需将输入流的偏移量设置回运行 `p` 之前的位置即可。但现在消耗量跟偏移量会不匹配，但在这里没有关系，因为我们调用 `fail` 立即结束了语法分析。但这在其它地方可能会出问题，我们将在后面的章节中看到如何改进。

## 表达式的语法分析

「表达式」是指由一些项和应用于这些项的运算符组成的结构。运算符可以前置、中置、后置，可以左结合、右结合，可以有不同的优先级。这种构造的一个例子是学校里教的算术表达式：

```
a * (b + 2)
```

这里我们可以看到两种不同的项：变量（`a`、`b`）和整数（`2`）。另外还有两种运算符：`*` 和 `+`。

为表达式编写一个正确的语法分析器大概需要假以时日。为此，[parser-combinators](https://hackage.haskell.org/package/parser-combinators) 包提供了 `Control.Monad.Combinators.Expr` 模块，它一共导出了两样东西：`Operator` 数据类型和 `makeExprParser` 工具函数。两者均文档齐全，所以本节我们不会复述文档，而是编写一个简单但功能完备的表达式语法分析器。

让我们先定义一个表示抽象语法树的数据结构：

```haskell
data Expr
  = Var String
  | Int Int
  | Negation Expr
  | Sum      Expr Expr
  | Subtr    Expr Expr
  | Product  Expr Expr
  | Division Expr Expr
  deriving (Eq, Ord, Show)
```

要用 `makeExprParser` 我们得给它一个项语法分析器和一个运算符表：

```haskell
makeExprParser :: MonadParsec e s m
  => m a               -- ^ Term parser
  -> [[Operator m a]]  -- ^ Operator table, see 'Operator'
  -> m a               -- ^ Resulting expression parser
```

让我们从项语法分析器开始。我们可以把项视为一个盒子，当处理结合性和优先级之类的东西时，表达式的语法分析算法会将其视为不可分割的整体。在我们例子中，有三类东西属于项：变量、整数和括号中的整个表达式。沿用前面几章节的定义，我们可以把项语法分析器定义为：

```haskell
pVariable :: Parser Expr
pVariable = Var <$> lexeme
  ((:) <$> letterChar <*> many alphaNumChar <?> "variable")

pInteger :: Parser Expr
pInteger = Int <$> lexeme L.decimal

parens :: Parser a -> Parser a
parens = between (symbol "(") (symbol ")")

pTerm :: Parser Expr
pTerm = choice
  [ parens pExpr
  , pVariable
  , pInteger
  ]

pExpr :: Parser Expr
pExpr = makeExprParser pTerm operatorTable

operatorTable :: [[Operator Parser Expr]]
operatorTable = undefined -- TODO
```

`pVariable`、`pInteger` 和 `parens` 的定义应该没什么疑问。这里幸运的是我们不需要在 `pTerm` 中使用 `try`，因为项的语法没有重叠之处：

- 如果我们看到左括号 `(`，那紧接着肯定是一个表达式；
- 如果我们看到一个字母，那肯定是标识符的开始；
- 如果我们看到一个数字，那肯定是整数的开始。

最后，为了完成 `pExpr`，我们需要定义 `operatorTable`，从类型可以看出它是个嵌套列表。每个内层列表装着相同优先级的运算符，而整个外层列表以优先级降序排列。一组运算符的优先级越高，它们结合得就越紧。

```haskell
data Operator m a -- N.B.
  = InfixN  (m (a -> a -> a)) -- ^ Non-associative infix
  | InfixL  (m (a -> a -> a)) -- ^ Left-associative infix
  | InfixR  (m (a -> a -> a)) -- ^ Right-associative infix
  | Prefix  (m (a -> a))      -- ^ Prefix
  | Postfix (m (a -> a))      -- ^ Postfix

operatorTable :: [[Operator Parser Expr]]
operatorTable =
  [ [ prefix "-" Negation
    , prefix "+" id
    ]
  , [ binary "*" Product
    , binary "/" Division
    ]
  , [ binary "+" Sum
    , binary "-" Subtr
    ]
  ]

binary :: Text -> (Expr -> Expr -> Expr) -> Operator Parser Expr
binary  name f = InfixL  (f <$ symbol name)

prefix, postfix :: Text -> (Expr -> Expr) -> Operator Parser Expr
prefix  name f = Prefix  (f <$ symbol name)
postfix name f = Postfix (f <$ symbol name)
```

注意 `binary` 中 `InfixL` 接受的 `Parser (Expr -> Expr -> Expr)` 我们是怎么写的，相似的还有 `prefix` 和 `postfix` 中的 `Parser (Expr -> Expr)`。也就是说，我们先运行 `symbol name` 然后返回一个函数，它会依次接受各项作为参数并返回 `Expr` 类型的结果。

准备好了，现在可以试试我们的语法分析器了！

```
λ> parseTest (pExpr <* eof) "a * (b + 2)"
Product (Var "a") (Sum (Var "b") (Int 2))
```
```
λ> parseTest (pExpr <* eof) "a * b + 2"
Sum (Product (Var "a") (Var "b")) (Int 2)
```
```
λ> parseTest (pExpr <* eof) "a * b / 2"
Division (Product (Var "a") (Var "b")) (Int 2)
```
```
λ> parseTest (pExpr <* eof) "a * (b $ 2)"
1:8:
  |
1 | a * (b $ 2)
  |        ^
unexpected '$'
expecting ')' or operator
```

`Control.Monad.Combinators.Expr` 模块的文档里有一些提示，在不太标准的情况下很有用，最好也读一下。

## 缩进敏感的语法分析

`Text.Megaparsec.Char.Lexer` 模块还包含一些工具，在处理对缩进敏感的语法时很有用。我们会先综述一下可用的组合子，然后再把它们组装成一个对缩进敏感的语法分析器。

### `nonIndented` 和 `indentBlock`

让我们从最简单的 `nonIndented` 开始：

```haskell
nonIndented :: MonadParsec e s m
  => m ()              -- ^ How to consume indentation (white space)
  -> m a               -- ^ Inner parser
  -> m a
```

它允许内部语法分析器吃掉所有没缩进的输入，这是缩进敏感语法分析背后模型的一部分。我们规定，未缩进的部分是顶层定义，而所有缩进的部分直接或间接地从属于顶层定义。在 `megaparsec` 中，我们不需要任何额外的状态来表达这个想法。因为缩进是相对的，所以我们的想法是显式地把参考单词和缩进单词都传给语法分析器，这样就能通过纯的语法分析器组合来定义对缩进敏感的语法。

那么我们应当如何为缩进块定义语法分析器呢？让我们看一眼 `indentBlock` 的签名：

```haskell
indentBlock :: (MonadParsec e s m, Token s ~ Char)
  => m ()                -- ^ How to consume indentation (white space)
  -> m (IndentOpt m a b) -- ^ How to parse “reference” token
  -> m a
```

首先，我们指定如何吃掉缩进。要注意的是这里的空格消耗器必须也吃掉换行符，但正常来讲单词后面的换行符是不应该吃掉的。

如你所见，第二个参数允许我们对参考单词进行语法分析，并返回一个告诉 `indentBlock` 接下来做什么的数据结构。下面是几种选择：

```haskell
data IndentOpt m a b
  = IndentNone a
    -- ^ Parse no indented tokens, just return the value
  | IndentMany (Maybe Pos) ([b] -> m a) (m b)
    -- ^ Parse many indented tokens (possibly zero), use given indentation level
    -- (if 'Nothing', use level of the first indented token);
    -- the second argument tells how to get the final result, and
    -- the third argument describes how to parse an indented token
  | IndentSome (Maybe Pos) ([b] -> m a) (m b)
    -- ^ Just like 'IndentMany', but requires at least one indented token to be present
```

我们可以改变主意不对缩进单词进行语法分析，也可以处理许多缩进单词。我们可以让 `indentBlock` 检测首个缩进单词的缩进层级并使用它，也可以手动指定缩进层级。

### 简单的缩进列表

让我们试着对一个简单的缩进列表进行语法分析，我们从导入部分开始：

```haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TupleSections     #-}

module Main (main) where

import Control.Applicative
import Control.Monad (void)
import Data.Text (Text)
import Data.Void
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Text.Megaparsec.Char.Lexer as L

type Parser = Parsec Void Text
```

我们需要两种空格消耗器：一种 `scn` 会吃掉换行符，另一种 `sc` 不会（实际上在这里它只处理空格和制表符）：

```haskell
lineComment :: Parser ()
lineComment = L.skipLineComment "#"

scn :: Parser ()
scn = L.space space1 lineComment empty

sc :: Parser ()
sc = L.space (void $ some (char ' ' <|> char '\t')) lineComment empty

lexeme :: Parser a -> Parser a
lexeme = L.lexeme sc
```

为了好玩，我们还允许 `#` 开头的行注释。

`pItemList` 是顶层形式，它包括参考单词（表头）和缩进单词（表项）：

```haskell
pItemList :: Parser (String, [String]) -- header and list items
pItemList = L.nonIndented scn (L.indentBlock scn p)
  where
    p = do
      header <- pItem
      return (L.IndentMany Nothing (return . (header, )) pItem)
```

对于我们来讲，表项就是一串字母、数字和短横线组成的序列：

```haskell
pItem :: Parser String
pItem = lexeme (some (alphaNumChar <|> char '-')) <?> "list item"
```

让我们将代码载入到 GHCi，用内置的 `parseTest` 试试：

```
λ> parseTest (pItemList <* eof) ""
1:1:
  |
1 | <empty line>
  | ^
unexpected end of input
expecting list item
```
```
λ> parseTest (pItemList <* eof) "something"
("something",[])
```
```
λ> parseTest (pItemList <* eof) "  something"
1:3:
  |
1 |   something
  |   ^
incorrect indentation (got 3, should be equal to 1)
```
```
λ> parseTest (pItemList <* eof) "something\none\ntwo\nthree"
2:1:
  |
2 | one
  | ^
unexpected 'o'
expecting end of input
```

记住我们用的是 `IndentMany` 选项，所以空列表是可以的。另一方面，内置的 `space` 组合子已在错误信息中隐藏了「expecting more space」，所以现在的错误信息是完全合理的。

让我们继续试试：

```
λ> parseTest (pItemList <* eof) "something\n  one\n    two\n  three"
3:5:
  |
3 |     two
  |     ^
incorrect indentation (got 5, should be equal to 3)
```
```
λ> parseTest (pItemList <* eof) "something\n  one\n  two\n three"
4:2:
  |
4 |  three
  |  ^
incorrect indentation (got 2, should be equal to 3)
```
```
λ> parseTest (pItemList <* eof) "something\n  one\n  two\n  three"
("something",["one","two","three"])
```

让我们把 `IndentMany` 换成 `IndentSome`，把 `Nothing` 换成 `Just (mkPos 5)`（缩进层级从 1 开始数，所以这表示需要 4 个空格的缩进）：

```haskell
pItemList :: Parser (String, [String])
pItemList = L.nonIndented scn (L.indentBlock scn p)
  where
    p = do
      header <- pItem
      return (L.IndentSome (Just (mkPos 5)) (return . (header, )) pItem)
```

现在：

```
λ> parseTest (pItemList <* eof) "something\n"
2:1:
  |
2 | <empty line>
  | ^
incorrect indentation (got 1, should be greater than 1)
```
```
λ> parseTest (pItemList <* eof) "something\n  one"
2:3:
  |
2 |   one
  |   ^
incorrect indentation (got 3, should be equal to 5)
```
```
λ> parseTest (pItemList <* eof) "something\n    one"
("something",["one"])
```

第一条错误信息可能有点令人惊讶，但 `megaparsec` 知道列表里至少得有一项，所以它检查了缩进层级发现是 1，于是报告了错误。

### 嵌套缩进列表

让我们允许表项拥有子项，为此我们创建了一个新的语法分析器 `pComplexItem`：

```haskell
pComplexItem :: Parser (String, [String])
pComplexItem = L.indentBlock scn p
  where
    p = do
      header <- pItem
      return (L.IndentMany Nothing (return . (header, )) pItem)

pItemList :: Parser (String, [(String, [String])])
pItemList = L.nonIndented scn (L.indentBlock scn p)
  where
    p = do
      header <- pItem
      return (L.IndentSome Nothing (return . (header, )) pComplexItem)
```

如果我们把下面这样的列表喂进去：

```
first-chapter
  paragraph-one
      note-A # an important note here!
      note-B
  paragraph-two
    note-1
    note-2
  paragraph-three
```

那我们的语法分析器会返回：

```
Right
  ( "first-chapter"
  , [ ("paragraph-one",   ["note-A","note-B"])
    , ("paragraph-two",   ["note-1","note-2"])
    , ("paragraph-three", [])
    ]
  )
```

以上演示了这个方法是如何扩展到嵌套缩进结构上的，我们并没有引入额外的状态。

### 加入折行

「折行」可以包含多行元素，不过后续元素的缩进层级必须高于首个元素。

让我们来试用一下 `lineFold`：

```haskell
pComplexItem :: Parser (String, [String])
pComplexItem = L.indentBlock scn p
  where
    p = do
      header <- pItem
      return (L.IndentMany Nothing (return . (header, )) pLineFold)

pLineFold :: Parser String
pLineFold = L.lineFold scn $ \sc' ->
  let ps = some (alphaNumChar <|> char '-') `sepBy1` try sc'
  in unwords <$> ps <* scn -- (1)
```

`lineFold` 的工作方式是：我们先给它一个接受换行符的空格消耗器 `scn`，然后它还回来一个特殊的空格消耗器 `sc'`，让我们能够在回调中吃掉折行元素之间的空格。

为什么 (1) 处要用 `try sc'` 和 `scn` 呢？情况是这样的：

- 折行元素只会比首个元素缩进更多。
- `sc'` 吃掉空格（也会吃换行符）之后，该列应该比起始列更大。
- 相反，如果吃掉空格后该列小于等于起始列，`sc'` 会停下来。它失败时不会吃掉任何输入（感谢 `try`），`scn` 会被用来挑选空格。
- 前面使用的 `sc'` 已利用会吃换行符的空格消耗器来探测空格，所以它逻辑上也会在挑选尾随空格时吃掉换行符。这就是为什么我们在 (1) 处用 `scn` 而不用 `sc`。

【练习】我们语法分析器的最终版本留给读者做测试。你可以创建多个折行元素，语法分析之后它们会用一个空格拼接在一起。

## 编写高效的语法分析器

让我们讨论一下怎么才能提高 `megaparsec` 语法分析器的性能。不过首先要指出，我们应当用性能分析和基准测试来验证我们的改进。这是我们在性能调优时检查是否有效的唯一方法。

这里有一些常见的建议：

- 如果你的语法分析器用的是单子栈而非普通的 `Parsec` 单子（回忆一下，这是使用 `Identity` 的 `ParsecT` 单子变换子，非常轻量），请确保 `transformers` 库的版本不低于 0.5，`megaparsec` 的版本不低于 7.0。这两个库在上述版本均有关键性的性能提升，只要升级就能变快。
- `Parsec` 单子总是比基于 `ParsecT` 的单子变换子更快。除非绝对必要，请避免使用 `StateT`、`WriterT` 或者其它单子变换子。往单子栈里加得越多，语法分析就越慢。
- 回溯是个代价高昂的操作。请避免构建冗长的选择链，其中的每个选择都有可能在失败前陷得很深。
- 除非确实有理由，请避免让语法分析器保持多态。最好指定一下语法分析器的具体类型，比如 `type Parser = Parsec Void Text`。这样能让 GHC 更好地进行优化。
- 尽可能内联（当然，是在合理的地方）。内联的巨大作用可能会令你难以置信，特别是对于那些很短的函数。这对跨模块使用的语法分析器尤其有用，因为 `INLINE` 和 `INLINEABLE` 编译指令能让 GHC 把函数定义转储到接口文件，这有助于进行特化。
- 尽可能使用快速的原语，比如 `takeWhileP`、`takeWhile1P` 和 `takeP`。[这篇博客](https://markkarpov.com/post/megaparsec-more-speed-more-power.html#there-is-hope)解释了为什么它们这么快。
- 尽可能使用 `satisfy` 和 `notChar`，而不要使用 `oneOf` 和 `noneOf`。

尽量上面大多数建议不需要进一步解释，但我觉得最好养成习惯使用这三个新的原语：`takeWhileP`、`takeWhile1P` 和 `takeP`。前两个尤其常见，能帮我们替换掉一些基于 `many` 和 `some` 的结构，它们更快并且会改而返回一大块输入流，也就是我们之前说的 `Tokens s` 类型。

举例来说，回忆一下我们对 URI 的用户名进行语法分析时用到下面的代码：

```haskell
  user <- T.pack <$> some alphaNumChar
```

我们可以把它替换为 `takeWhile1P`：

```haskell
  user <- takeWhile1P (Just "alpha num character") isAlphaNum
  --                  ^                            ^
  --                  |                            |
  -- label for tokens we match against         predicate
```

当我们对 `ByteString` 和 `Text` 进行语法分析时，这会比原来的方法快很多。顺便注意一下，我们能从 `takeWhile1P` 直接拿到 `Text`，所以就不再需要 `T.pack` 了。

下面这些等式对于理解 `takeWhileP` 和 `takeWhile1P` 的 `Maybe String` 参数很有帮助：

```haskell
takeWhileP  (Just "foo") f = many (satisfy f <?> "foo")
takeWhileP  Nothing      f = many (satisfy f)
takeWhile1P (Just "foo") f = some (satisfy f <?> "foo")
takeWhile1P Nothing      f = some (satisfy f)
```

## 语法分析错误

到现在我们已经探索了 `megaparsec` 的大多数特性，是时候来学习一下语法分析错误了：它们如何定义、如何触发、如何在运行时处理它们。

### 错误的定义

`ParseError` 类型有如下定义：

```haskell
data ParseError s e
  = TrivialError Int (Maybe (ErrorItem (Token s))) (Set (ErrorItem (Token s)))
    -- ^ Trivial errors, generated by Megaparsec's machinery. 
    -- The data constructor includes the offset of error, unexpected token (if any), and expected tokens.
  | FancyError Int (Set (ErrorFancy e))
    -- ^ Fancy, custom errors.
```

用中文来讲：`ParseError` 要么是个 `TrivialError` 要么是个 `FancyError`，前者会提供偏移量信息、不期而遇的单词（一个或没有）和我们期待的单词集合（可能为空）。

`ParseError s e` 有下面两个类型参数：

- `s` 是输入流的类型；
- `e` 是自定义错误的类型。

`ErrorItem` 是这样定义的：

```haskell
data ErrorItem t
  = Tokens (NonEmpty t)      -- ^ Non-empty stream of tokens
  | Label (NonEmpty Char)    -- ^ Label (cannot be empty)
  | EndOfInput               -- ^ End of input
```

还有 `ErrorFancy`：

```haskell
data ErrorFancy e
  = ErrorFail String
    -- ^ 'fail' has been used in parser monad
  | ErrorIndentation Ordering Pos Pos
    -- ^ Incorrect indentation error:
    --     desired ordering between reference level and actual level,
    --     reference indentation level,
    --     actual indentation level
  | ErrorCustom e
    -- ^ Custom error data, can be conveniently disabled by indexing 'ErrorFancy' by 'Void'
```

`ErrorFancy` 包括两个 `megaparsec` 常见错误的数据构造器：

- `fail` 函数会让语法分析器失败并报告任意 `String`；
- 因为该库原生支持对缩进敏感的语法，所以错误类型也能方便地存储缩进相关信息。

最后，`ErrorCustom` 是允许将任意数据嵌入 `ErrorFancy` 类型的「扩展槽」。如果我们不需要在语法分析错误中使用自定义数据，我们可以把 `Void` 传给 `ErrorFancy`。由于 `Void` 不接受非底类型的值，`ErrorCustom` 就相当于「取消」了，用抽象数据类型做类比的话，就是「与零的积」。

在旧版本中，`ParseError` 会直接被 `parse` 等函数返回，但版本 7.0 推迟了每个错误的行和列的计算，以及用于显示错误的相关行内容的获取。这能让语法分析更快，因为这些信息通常只有在语法分析失败时才有用。另一个旧版本的问题是，同时显示多个错误需要每次重新遍历输入来获取正确的行。

这个问题现在由 `ParseErrorBundle` 数据类型解决了：

```haskell
-- | A non-empty collection of 'ParseError's equipped with 'PosState' that
-- allows to pretty-print the errors efficiently and correctly.

data ParseErrorBundle s e = ParseErrorBundle
  { bundleErrors :: NonEmpty (ParseError s e)
    -- ^ A collection of 'ParseError's that is sorted by parse error offsets
  , bundlePosState :: PosState s
    -- ^ State that is used for line\/column calculation
  }
```

所有运行语法分析的函数都会返回 `ParseErrorBundle`，里面会有设置好的 `bundlePosState` 和 `ParseError`。里面的 `ParseError` 列表可以由用户自行扩展，不过这样得由用户来保证它们仍按照偏移量有序排列。

### 如何触发错误

让我们讨论一下触发语法分析错误的几种不同方式，最简单的是 `fail` 函数：

```
λ> parseTest (fail "I'm failing, help me!" :: Parser ()) ""
1:1:
  |
1 | <empty line>
  | ^
I'm failing, help me!
```

对于很多熟悉其它简单的语法分析库（比如 `parsec`）的人来讲，这通常已经足够了。然而，除了向用户显示语法分析错误之外，我们还有可能需要分析或是处理它，这时候 `String` 就不是很方便了。

平凡的语法分析错误通常都是 `megaparsec` 生成的，但我们也能自己用 `failure` 组合子触发这样的错误：

```haskell
failure :: MonadParsec e s m
  => Maybe (ErrorItem (Token s)) -- ^ Unexpected item (if any)
  -> Set (ErrorItem (Token s)) -- ^ Expected items
  -> m a
unfortunateParser :: Parser ()
unfortunateParser = failure (Just EndOfInput) (Set.fromList es)
  where
    es = [Tokens (NE.fromList "a"), Tokens (NE.fromList "b")]
```
```
λ> parseTest unfortunateParser ""
1:1:
  |
1 | <empty line>
  | ^
unexpected end of input
expecting 'a' or 'b'
```

跟基于 `fail` 的方法不同，平凡的错误很容易进行模型匹配，或是审视和修改。

对于花哨的错误，相应地我们有 `fancyFailure` 组合子：

```haskell
fancyFailure :: MonadParsec e s m
  => Set (ErrorFancy e) -- ^ Fancy error components
  -> m a
```

但对于 `fancyFailure`，我们通常会去定义一个工具函数，而不是直接调用 `fancyFailure`：

```haskell
incorrectIndent :: MonadParsec e s m
  => Ordering  -- ^ Desired ordering between reference level and actual level
  -> Pos               -- ^ Reference indentation level
  -> Pos               -- ^ Actual indentation level
  -> m a
incorrectIndent ord ref actual = fancyFailure . E.singleton $
  ErrorIndentation ord ref actual
```

作为添加自定义语法分析错误组件的例子，让我们创建这样一个特殊的语法分析错误，它会报告给定的 `Text` 值不是关键字。

首先，我们需要定义一个数据类型，其构造器代表我们想要支持的场景：

```haskell
data Custom = NotKeyword Text
  deriving (Eq, Show, Ord)
```

并告诉 `megaparsec` 如何显示这个错误：

```haskell
instance ShowErrorComponent Custom where
  showErrorComponent (NotKeyword txt) = T.unpack txt ++ " is not a keyword"
```

接下来我们更新一下我们的 `Parser` 别名：

```haskell
type Parser = Parsec Custom Text
```

之后我们定义一个 `notKeyword` 工具函数：

```haskell
notKeyword :: Text -> Parser a
notKeyword = customFailure . NotKeyword
```

其中 `customFailure` 是来自 `Text.Megaparsec` 模块的工具函数：

```haskell
customFailure :: MonadParsec e s m => e -> m a
customFailure = fancyFailure . E.singleton . ErrorCustom
```

最后，让我们试一下：

```
λ> parseTest (notKeyword "foo" :: Parser ()) ""
1:1:
  |
1 | <empty line>
  | ^
foo is not a keyword
```

### 显示错误

显示 `ParseErrorBundle` 可以用 `errorBundlePretty` 函数完成：

```haskell
-- | Pretty-print a 'ParseErrorBundle'. All 'ParseError's in the bundle will
-- be pretty-printed in order together with the corresponding offending
-- lines by doing a single efficient pass over the input stream. The
-- rendered 'String' always ends with a newline.

errorBundlePretty
  :: ( Stream s
     , ShowErrorComponent e
     )
  => ParseErrorBundle s e -- ^ Parse error bundle to display
  -> String               -- ^ Textual rendition of the bundle
```

99% 的情况下你只需要这么一个函数。

### 在运行时接住错误

`megaparsec` 另一个有用的特性是它能够「接住」语法分析错误，并以某种方式改变它，然后再重新抛出错误，就像异常一样。这可以用 `observing` 原语实现：

```haskell
-- | @'observing' p@ allows to “observe” failure of the @p@ parser, should
-- it happen, without actually ending parsing, but instead getting the
-- 'ParseError' in 'Left'. On success parsed value is returned in 'Right'
-- as usual. Note that this primitive just allows you to observe parse
-- errors as they happen, it does not backtrack or change how the @p@
-- parser works in any way.

observing :: MonadParsec e s m
  => m a             -- ^ The parser to run
  -> m (Either (ParseError (Token s) e) a)
```

下面是演示 `observing` 典型用法的完整程序：

```haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeApplications  #-}

module Main (main) where

import Control.Applicative
import Data.List (intercalate)
import Data.Set (Set)
import Data.Text (Text)
import Data.Void
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Data.Set as Set

data Custom
  = TrivialWithLocation
    [String] -- position stack
    (Maybe (ErrorItem Char))
    (Set (ErrorItem Char))
  | FancyWithLocation
    [String] -- position stack
    (ErrorFancy Void) -- Void, because we do not want to allow to nest Customs
  deriving (Eq, Ord, Show)

instance ShowErrorComponent Custom where
  showErrorComponent (TrivialWithLocation stack us es) =
    parseErrorTextPretty (TrivialError @Char @Void undefined us es)
      ++ showPosStack stack
  showErrorComponent (FancyWithLocation stack cs) =
    parseErrorTextPretty (FancyError @Text @Void undefined (Set.singleton cs))
      ++ showPosStack stack

showPosStack :: [String] -> String
showPosStack = intercalate ", " . fmap ("in " ++)

type Parser = Parsec Custom Text

inside :: String -> Parser a -> Parser a
inside location p = do
  r <- observing p
  case r of
    Left (TrivialError _ us es) ->
      fancyFailure . Set.singleton . ErrorCustom $
        TrivialWithLocation [location] us es
    Left (FancyError _ xs) -> do
      let f (ErrorFail msg) = ErrorCustom $
            FancyWithLocation [location] (ErrorFail msg)
          f (ErrorIndentation ord rlvl alvl) = ErrorCustom $
            FancyWithLocation [location] (ErrorIndentation ord rlvl alvl)
          f (ErrorCustom (TrivialWithLocation ps us es)) = ErrorCustom $
            TrivialWithLocation (location:ps) us es
          f (ErrorCustom (FancyWithLocation ps cs)) = ErrorCustom $
            FancyWithLocation (location:ps) cs
      fancyFailure (Set.map f xs)
    Right x -> return x

myParser :: Parser String
myParser = some (char 'a') *> some (char 'b')

main :: IO ()
main = do
  parseTest (inside "foo" myParser) "aaacc"
  parseTest (inside "foo" $ inside "bar" myParser) "aaacc"
```

【练习】深入理解这个程序是如何工作的。

如果运行这个程序，会看到以下输出：

```
1:4:
  |
1 | aaacc
  |    ^
unexpected 'c'
expecting 'a' or 'b'
in foo
1:4:
  |
1 | aaacc
  |    ^
unexpected 'c'
expecting 'a' or 'b'
in foo, in bar
```

因此，这个特性可以用来给语法分析错误附加位置标签，或是定义能以某种方式处理该错误的「区域」。这种惯用法很有用，所以甚至有一个基于 `observing` 定义的工具函数 `region`：

```haskell
-- | Specify how to process 'ParseError's that happen inside of this
-- wrapper. This applies to both normal and delayed 'ParseError's.
--
-- As a side-effect of the implementation the inner computation will start
-- with empty collection of delayed errors and they will be updated and
-- “restored” on the way out of 'region'.

region :: MonadParsec e s m
  => (ParseError s e -> ParseError s e) -- ^ How to process 'ParseError's
  -> m a                 -- ^ The “region” that the processing applies to
  -> m a
region f m = do
  r <- observing m
  case r of
    Left err -> parseError (f err) -- see the next section
    Right x -> return x
```

【练习】用 `region` 重写之前程序中的 `inside` 函数。

### 控制错误的位置

`region` 的定义使用了 `parseError` 原语：

```haskell
parseError :: MonadParsec e s m => ParseError s e -> m a
```

这是错误报告的基础原语，我们目前见到的所有其它函数都基于 `parseError` 定义的：

```haskell
failure
  :: MonadParsec e s m
  => Maybe (ErrorItem (Token s)) -- ^ Unexpected item (if any)
  -> Set (ErrorItem (Token s)) -- ^ Expected items
  -> m a
failure us ps = do
  o <- getOffset
  parseError (TrivialError o us ps)
```
```haskell
fancyFailure
  :: MonadParsec e s m
  => Set (ErrorFancy e) -- ^ Fancy error components
  -> m a
fancyFailure xs = do
  o <- getOffset
  parseError (FancyError o xs)
```

`parseError` 可以让你设置错误的偏移量（也就是位置），而不必是输入流的当前位置。让我们回到很久之前的那个例子：

```haskell
withPredicate2
  :: (a -> Bool)       -- ^ The check to perform on parsed input
  -> String            -- ^ Message to print when the check fails
  -> Parser a          -- ^ Parser to run
  -> Parser a          -- ^ Resulting parser that performs the check
withPredicate2 f msg p = do
  o <- getOffset
  r <- p
  if f r
    then return r
    else do
      setOffset o
      fail msg
```

我们注意到 `setOffset o` 能让错误被正确定位，但它的副作用是会使语法分析状态失效，也就是说偏移量不再反映现实情况了。在更复杂的语法分析器中，这可能会是个现实的问题。举例来说，想象一下你用 `observing` 包住了 `withPredicate2`，那么 `fail` 之后可能还会有代码运行。

有了 `parseError` 和 `region`，我们能够正确地解决这个问题了：要么使用 `parseError` 来重设错误位置，要么直接用 `region`：

```haskell
withPredicate3
  :: (a -> Bool)       -- ^ The check to perform on parsed input
  -> String            -- ^ Message to print when the check fails
  -> Parser a          -- ^ Parser to run
  -> Parser a          -- ^ Resulting parser that performs the check
withPredicate3 f msg p = do
  o <- getOffset
  r <- p
  if f r
    then return r
    else region (setErrorOffset o) (fail msg)
```
```haskell
withPredicate4
  :: (a -> Bool)       -- ^ The check to perform on parsed input
  -> String            -- ^ Message to print when the check fails
  -> Parser a          -- ^ Parser to run
  -> Parser a          -- ^ Resulting parser that performs the check
withPredicate4 f msg p = do
  o <- getOffset
  r <- p
  if f r
    then return r
    else parseError (FancyError o (Set.singleton (ErrorFail msg)))
```

### 报告多个错误

最后，`megaparsec` 允许我们在一次运行过程中触发多个语法分析错误。这能帮助我们一次修复多处错误，而不需要运行好几次语法分析器。

拥有多错误语法分析器的前提条件是，它要能跳过一部分有问题的输入，并从一个已知没问题的位置继续进行语法分析。这部分工作要用 `withRecovery` 原语完成：

```haskell
-- | @'withRecovery' r p@ allows continue parsing even if parser @p@
-- fails. In this case @r@ is called with the actual 'ParseError' as its
-- argument. Typical usage is to return a value signifying failure to
-- parse this particular object and to consume some part of the input up
-- to the point where the next object starts.
--
-- Note that if @r@ fails, original error message is reported as if
-- without 'withRecovery'. In no way recovering parser @r@ can influence
-- error messages.

withRecovery
  :: (ParseError s e -> m a) -- ^ How to recover from failure
  -> m a             -- ^ Original parser
  -> m a             -- ^ Parser that can recover from failures
```

在 Megaparsec 8 之前，`a` 必须是包含成功和失败两种可能性的和类型，比如说 `Either (ParseError s e) Result`。语法分析错误在收集后会加入 `ParseErrorBundle` 以进行显示。不必说，这些都是对用户不友好的高级用法。

Megaparsec 8 支持了「延迟错误」：

```haskell
-- | Register a 'ParseError' for later reporting. This action does not end
-- parsing and has no effect except for adding the given 'ParseError' to the
-- collection of “delayed” 'ParseError's which will be taken into
-- consideration at the end of parsing. Only if this collection is empty
-- parser will succeed. This is the main way to report several parse errors
-- at once.

registerParseError :: MonadParsec e s m => ParseError s e -> m ()

-- | Like 'failure', but for delayed 'ParseError's.

registerFailure
  :: MonadParsec e s m
  => Maybe (ErrorItem (Token s)) -- ^ Unexpected item (if any)
  -> Set (ErrorItem (Token s)) -- ^ Expected items
  -> m ()

-- | Like 'fancyFailure', but for delayed 'ParseError's.

registerFancyFailure
  :: MonadParsec e s m
  => Set (ErrorFancy e) -- ^ Fancy error components
  -> m ()
```

这些错误可以在 `withRecovery` 的错误处理回调中注册，所以结果类型会是 `Maybe Result`。这样可以把延迟错误列入最后的 `ParseErrorBundle`，并且在错误列表非空的情况让语法分析失败。

有了这些，我们希望编写多错误语法分析器的做法会在用户群中更加普遍。

## 测试 Megaparsec 语法分析器

对语法分析器进行测试是大多数人迟早要面对的事情，所以我们有义务提一下。最推荐的方式是使用 [hspec-megaparsec](https://hackage.haskell.org/package/hspec-megaparsec) 包，里面有一些效用期望，比如 `shouldParse`、`parseSatisfies` 等等，能和 `hspec` 测试框架协同工作。

让我们从一个用例开始：

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main (main) where

import Control.Applicative
import Data.Text (Text)
import Data.Void
import Test.Hspec
import Test.Hspec.Megaparsec
import Text.Megaparsec
import Text.Megaparsec.Char

type Parser = Parsec Void Text

myParser :: Parser String
myParser = some (char 'a')

main :: IO ()
main = hspec $
  describe "myParser" $ do
    it "returns correct result" $
      parse myParser "" "aaa" `shouldParse` "aaa"
    it "result of parsing satisfies what it should" $
      parse myParser "" "aaaa" `parseSatisfies` ((== 4) . length)
```

`shouldParse` 接受 `Either (ParseErrorBundle s e) a`，即语法分析的结果和一个用来进行比较的 `a` 类型的值，这可能是用得最多的工具函数。`parseSatisfies` 跟它很相似，但不是跟期待的结果比较是否相等，而是用任意断言检查结果。

其它简单的效用期望还有 `shouldSucceedOn` 和 `shouldFailOn`（但很少用到它们）：

```haskell
    it "should parse 'a's all right" $
      parse myParser "" `shouldSucceedOn` "aaaa"
    it "should fail on 'b's" $
      parse myParser "" `shouldFailOn` "bbb"
```

在使用 `megaparsec` 时，我们想要让语法分析错误更加精确。为了测试语法分析错误我们可以使用 `shouldFailWith`，用法如下：

```haskell
    it "fails on 'b's producing correct error message" $
      parse myParser "" "bbb" `shouldFailWith`
        TrivialError
          0
          (Just (Tokens ('b' :| [])))
          (Set.singleton (Tokens ('a' :| [])))
```

像这样写出 `TrivialError` 挺让人厌烦的。`ParseError` 的定义包含了像 `Set` 和 `NonEmpty` 这样「不方便」的类型，就像我们上面见到的那样，写起来很麻烦。幸运的是，`Test.Hspec.Megaparsec` 也重新导出了 `Text.Megaparsec.Error.Builder` 模块，里面提供了更方便地构建 `ParseError` 的 API。让我们来看看 `err`：

```haskell
    it "fails on 'b's producing correct error message" $
      parse myParser "" "bbb" `shouldFailWith` err 0 (utok 'b' <> etok 'a')
```

- `err` 的第一个参数是错误的偏移量（在出错之前我们吃掉了多少单词），这里它就是 0。
- `utok` 表示「不期而遇的单词」，类似地 `etok` 表示「我们期待的单词」。

【练习】要构建花哨的错误，也有类似的工具函数叫做 `errFancy`，请了解一下。

最后，还可以用 `failsLeaving` 和 `succeedLeaving` 来测试输入的哪部分在语法分析后还没被吃掉：

```haskell
    it "consumes all 'a's but does not touch 'b's" $
      runParser' myParser (initialState "aaabbb") `succeedsLeaving` "bbb"
    it "fails without consuming anything" $
      runParser' myParser (initialState "bbbccc") `failsLeaving` "bbbccc"
```

这些函数应该用 `runParser'` 和 `runParserT'` 运行，因为它们支持自定义初始状态并且会返回最终状态（这就能检查输入流剩下的东西了）：

```haskell
runParser'
  :: Parsec e s a      -- ^ Parser to run
  -> State s           -- ^ Initial state
  -> (State s, Either (ParseError (Token s) e) a)
```
```haskell
runParserT' :: Monad m
  => ParsecT e s m a   -- ^ Parser to run
  -> State s           -- ^ Initial state
  -> m (State s, Either (ParseError (Token s) e) a)
```

`initialState` 函数接受输入流，返回该输入流构成的初始状态，而初始状态的其它记录字段会用默认值填充。

关于使用 `hspec-megaparsec`，下述代码会是你的灵感来源：

- `hspec-megaparsec` 编写的 [Megaparsec 自己的测试套件](https://github.com/mrkkrp/megaparsec/tree/master/megaparsec-tests/tests)；
- `hspec-megaparsec` 自带的[玩具测试套件](https://github.com/mrkkrp/hspec-megaparsec/blob/master/tests/Main.hs)。

## 使用自定义输入流

`megaparsec` 能用来对任何输入流进行语法分析，只要它是 `Stream` 类型类的实例。这意味着它可以和 `alex` 之类的词法分析工具配合使用。

为了不偏离我们的主题，我们不会展示 `alex` 是如何生成单词流的，我们就假定输入是下述形式：

```haskell
{-# LANGUAGE LambdaCase        #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE RecordWildCards   #-}
{-# LANGUAGE TypeFamilies      #-}

module Main (main) where

import Data.List.NonEmpty (NonEmpty (..))
import Data.Proxy
import Data.Void
import Text.Megaparsec
import qualified Data.List          as DL
import qualified Data.List.NonEmpty as NE
import qualified Data.Set           as Set

data MyToken
  = Int Int
  | Plus
  | Mul
  | Div
  | OpenParen
  | CloseParen
  deriving (Eq, Ord, Show)
```

为了报告语法分析错误，我们需要一种方式知道单词的起始位置、终止位置和长度，因此我们添加了 `WithPos`：

```haskell
data WithPos a = WithPos
  { startPos :: SourcePos
  , endPos :: SourcePos
  , tokenLength :: Int
  , tokenVal :: a
  } deriving (Eq, Ord, Show)
```

这下我们就有数据类型表示自己的流了：

```haskell
data MyStream = MyStream
  { myStreamInput :: String -- for showing offending lines
  , unMyStream :: [WithPos MyToken]
  }
```

接下来，我们需要让 `MyStream` 成为 `Stream` 类型类的实例。这需要 `TypeFamilies` 语言扩展，因为我们想要定义关联类型函数 `Token` 和 `Tokens`：

```haskell
instance Stream MyStream where
  type Token  MyStream = WithPos MyToken
  type Tokens MyStream = [WithPos MyToken]
  -- …
```

`Stream` 的文档可以在 `Text.Megaparsec.Stream` 模块中找到。现在我们直接把剩下的方法定义完：

```haskell
-- …
  tokenToChunk Proxy x = [x]
  tokensToChunk Proxy xs = xs
  chunkToTokens Proxy = id
  chunkLength Proxy = length
  chunkEmpty Proxy = null
  take1_ (MyStream _ []) = Nothing
  take1_ (MyStream str (t:ts)) = Just
    ( t
    , MyStream (drop (tokensLength pxy (t:|[])) str) ts
    )
  takeN_ n (MyStream str s)
    | n <= 0    = Just ([], MyStream str s)
    | null s    = Nothing
    | otherwise =
        let (x, s') = splitAt n s
        in case NE.nonEmpty x of
          Nothing -> Just (x, MyStream str s')
          Just nex -> Just (x, MyStream (drop (tokensLength pxy nex) str) s')
  takeWhile_ f (MyStream str s) =
    let (x, s') = DL.span f s
    in case NE.nonEmpty x of
      Nothing -> (x, MyStream str s')
      Just nex -> (x, MyStream (drop (tokensLength pxy nex) str) s')
  showTokens Proxy = DL.intercalate " "
    . NE.toList
    . fmap (showMyToken . tokenVal)
  tokensLength Proxy xs = sum (tokenLength <$> xs)
  reachOffset o PosState {..} =
    ( prefix ++ restOfLine
    , PosState
        { pstateInput = MyStream
            { myStreamInput = postStr
            , unMyStream = post
            }
        , pstateOffset = max pstateOffset o
        , pstateSourcePos = newSourcePos
        , pstateTabWidth = pstateTabWidth
        , pstateLinePrefix = prefix
        }
    )
    where
      prefix =
        if sameLine
          then pstateLinePrefix ++ preStr
          else preStr
      sameLine = sourceLine newSourcePos == sourceLine pstateSourcePos
      newSourcePos =
        case post of
          [] -> pstateSourcePos
          (x:_) -> startPos x
      (pre, post) = splitAt (o - pstateOffset) (unMyStream pstateInput)
      (preStr, postStr) = splitAt tokensConsumed (myStreamInput pstateInput)
      tokensConsumed =
        case NE.nonEmpty pre of
          Nothing -> 0
          Just nePre -> tokensLength pxy nePre
      restOfLine = takeWhile (/= '\n') postStr

pxy :: Proxy MyStream
pxy = Proxy

showMyToken :: MyToken -> String
showMyToken = \case
  (Int n)    -> show n
  Plus       -> "+"
  Mul        -> "*"
  Div        -> "/"
  OpenParen  -> "("
  CloseParen -> ")"
```

更多关于 `Stream` 类型类的背景资料（以及为什么它长这样）可以在[这篇博客](https://markkarpov.com/post/megaparsec-more-speed-more-power.html)中找到。

现在我们可以为自定义的流定义 `Parser` 了：

```haskell
type Parser = Parsec Void MyStream
```

下一步是基于 `token` 和 `tokens` 两个原语，定义基本的语法分析器了。对于原生支持的流我们有 `Text.Megaparsec.Byte` 和 `Text.Megaparsec.Char` 模块，但要使用自定义的单词，我们需要自定义工具函数。

```haskell
liftMyToken :: MyToken -> WithPos MyToken
liftMyToken myToken = WithPos pos pos 0 myToken
  where
    pos = initialPos ""

pToken :: MyToken -> Parser MyToken
pToken c = token test (Set.singleton . Tokens . nes . liftMyToken $ c)
  where
    test (WithPos _ _ _ x) =
      if x == c
        then Just x
        else Nothing
    nes x = x :| []

pInt :: Parser Int
pInt = token test Set.empty <?> "integer"
  where
    test (WithPos _ _ _ (Int n)) = Just n
    test _ = Nothing
```

最后让我们写一个语法分析器测试一下加法表达式：

```haskell
pSum = do
  a <- pInt
  _ <- pToken Plus
  b <- pInt
  return (a, b)
```

这里是一个样例输入：

```haskell
exampleStream :: MyStream
exampleStream = MyStream
  "5 + 6"
  [ at 1 1 (Int 5)
  , at 1 3 Div         -- (1)
  , at 1 5 (Int 6)
  ]
  where
    at  l c = WithPos (at' l c) (at' l (c + 1)) 2
    at' l c = SourcePos "" (mkPos l) (mkPos c)
```

让我们试一下：

```
λ> parseTest (pSum <* eof) exampleStream
(5,6)
```

如果我们把 (1) 处的 `Plus` 改成 `Div`，我们也能得到正确的错误信息：

```
λ> parseTest (pSum <* eof) exampleStream
1:3:
  |
1 | 5 + 6
  |   ^^
unexpected /
expecting +
```

换言之，我们拥有一个能够处理自定义流的功能完备的语法分析器了。
