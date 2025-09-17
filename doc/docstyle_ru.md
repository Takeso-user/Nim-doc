# Стиль документации

## Общие рекомендации

- См. также [nep1](nep1.html) — возможно, его стоит объединить с этим документом.
- Авторам рекомендуется документировать всё, что экспортируется; документация
  для приватных процедур тоже может быть полезна (видна при `nim doc --docInternal foo.nim`).
- В комментарии документации после каждого предложения (или фрагмента
  предложения) ставьте точку (`.`). Документация может состоять из одного
  фрагмента предложения, но если внутри блока несколько предложений, каждое
  последующее должно быть полным и в настоящем времени.
- Документация парсится как кастомный диалект reStructuredText (RST) с
  частичной поддержкой Markdown.
- В исходниках Nim предпочтительнее использовать одиночные обратные апострофы
  вместо двойных, это проще и поддерживается `nim doc`. Аналогично для RST
  файлов: `nim rst2html` отобразит их моноширинным шрифтом; добавление
  `.. default-role:: code` в RST-файл также сделает inline-код моноширинным
  при рендеринге, например, на GitHub.
- (спорно) В исходниках Nim для ссылок можно предпочесть синтаксис Markdown
  `[link text](link.html)` вместо RST-формы ``` link text<link.html>`_ `` —
  он проще и более распространён. `nim rst2html` также поддерживает его в RST.

```nim
proc someproc*(s: string, foo: int) =
  ## Use single backticks for inline code, e.g.: `s` or `someExpr(true)`.
  ## Use a backlash to follow with alphanumeric char: `int8`\s are great.
```

## Документация на уровне модуля

Документация модуля размещается в начале самого модуля. Каждая строка
документации начинается с двойного `##`. Иногда удобнее использовать
`##[ multiline docs containing code ]##`, см. `lib/pure/times.nim`.
Примерные образцы кода приветствуются и должны следовать общей RST-синтаксису:

````nim
## The `universe` module computes the answer to life, the universe, and everything.
##
##   ```
##   doAssert computeAnswerString() == 42
##   ```
````

В этом верхнеуровневом комментарии можно указывать авторство и копирайт,
они попадут в сгенерированную документацию.

```nim
## This is the best module ever. It provides answers to everything!
##
## :Author: Steve McQueen
## :Copyright: 1965
##
```

Оставляйте пустую строку между последней строкой верхнеуровневой документации
и началом Nim-кода (импорты, и т.д.).

## Процедуры, шаблоны, макросы, конвертеры и итераторы

Документация процедуры должна начинаться с заглавной буквы и быть в
настоящем времени. Переменные, упоминаемые в тексте, обрамляйте одиночными
обратными апострофами:

```nim
proc example1*(x: int) =
  ## Prints the value of `x`.
  echo x
```

Если пример использования поможет читателю, включите его в документацию
в формате RST как показано ниже.

````nim
proc addThree*(x, y, z: int8): int =
  ## Adds three `int8` values, treating them as unsigned and
  ## truncating the result.
  ##
  ##   ```
  ##   # things that aren't suitable for a `runnableExamples` go in code block:
  ##   echo execCmdEx("git pull")
  ##   drawOnScreen()
  ##   ```
  runnableExamples:
    # `runnableExamples` is usually preferred to code blocks, when possible.
    doAssert addThree(3, 125, 6) == -122
  result = x +% y +% z
````

Команда `nim doc` корректно подсветит Nim-код внутри блока документации.

## Типы

Экспортируемые типы тоже должны иметь документацию. Она может включать
примеры кода, но чаще такие примеры лучше помещать рядом с функциями,
к которым они относятся.

```nim
type
  NamedQueue*[T] = object ## Provides a linked data structure with names
                          ## throughout. It is named for convenience. I'm making
                          ## this comment long to show how you can, too.
    name*: string ## The name of the item
    val*: T ## Its value
    next*: ref NamedQueue[T] ## The next item in the queue
```

У вас есть некоторая свобода в размещении документации:

```nim
type
  NamedQueue*[T] = object
    ## Provides a linked data structure with names
    ## throughout. It is named for convenience. I'm making
    ## this comment long to show how you can, too.
    name*: string ## The name of the item
    val*: T ## Its value
    next*: ref NamedQueue[T] ## The next item in the queue
```

Убедитесь, что документация находится рядом с самим объектом (сбоку или
внутри объявления), иначе она может не стать частью основной документации.

```nim
type
  ## Bad: this documentation disappears because it annotates the `type` keyword
  ## above, not `NamedQueue`.
  NamedQueue*[T] = object
    name*: string ## This becomes the main documentation for the object, which
                  ## is not what we want.
    val*: T ## Its value
    next*: ref NamedQueue[T] ## The next item in the queue
```

## Var, Let и Const

При объявлении модульных констант и значений приветствуется документация.
Размещение комментариев аналогично разделу `type`.

```nim
const
  X* = 42 ## An awesome number.
  SpreadArray* = [
    [1,2,3],
    [2,3,1],
    [3,1,2],
  ] ## Doc comment for `SpreadArray`.
```

Размещение комментариев в других местах обычно допустимо, но такие
комментарии не попадут в сгенерированную документацию и потому должны
начинаться с одного `#`.

```nim
const
  BadMathVals* = [
    3.14, # pi
    2.72, # e
    0.58, # gamma
  ] ## A bunch of badly rounded values.
```

Nim поддерживает Unicode в комментариях, поэтому можно использовать символы
π, γ и т.п. прямо в комментариях:

```nim
const
  BadMathVals* = [
    3.14, # π
    2.72, # e
    0.58, # γ
  ] ## A bunch of badly rounded values (including π!).
```
