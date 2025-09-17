# Фильтры исходного кода

.. include:: rstcommon.rst
.. default-role:: code
.. contents::

Фильтр исходного кода (Source Code Filter, SCF) преобразует входной поток
символов во внутренний выходной поток до этапа парсинга. Фильтр может
использоваться для реализации шаблонизаторов или препроцессоров.

Чтобы применить фильтр к исходному файлу, используется обозначение `#?`:

    #? stdtmpl(subsChar = '$', metaChar = '#')
    #proc generateXML(name, age: string): string =
    #  result = ""
    <xml>
      <name>$name</name>
      <age>$age</age>
    </xml>

Как видно из примера, передача аргументов фильтру выполняется так же,
как при вызове обычной процедуры — с позиционными или именованными
параметрами. Набор доступных параметров зависит от конкретного фильтра.
До версии 0.12.0 вместо `#?` использовалось `#!`.

**Hint:** With `--hint:codeBegin:on`:option: or `--verbosity:2`:option:
(or higher) while compiling or `nim check`:cmd:, Nim lists the processed code after
each filter application.

## Usage

Сначала поместите код SCF в отдельный файл с описанием фильтров в первой
строке. Примечание: расширение файла может быть любым, но общепринятое
расширение — `.nimf` (ранее использовалось `.tmpl`, что было слишком
общим и мешало, например, GitHub распознавать файл как Nim исходник).

Если использовать приведённый ранее `generateXML` и назвать файл
`xmlGen.nimf`, то в `main.nim` можно подключить его так:

```nim
include "xmlGen.nimf"

echo generateXML("John Smith","42")
```

## Pipe operator

Фильтры можно комбинировать с оператором `|` (pipe):

    #? strip(startswith="<") | stdtmpl
    #proc generateXML(name, age: string): string =
    #  result = ""
    <xml>
      <name>$name</name>
      <age>$age</age>
    </xml>

## Available filters

### Replace filter

Фильтр replace заменяет подстроки в каждой строке.

Параметры и их значения по умолчанию:

- `sub: string = ""`
  : подстрока, которую ищут

- `by: string = ""`
  : строка, на которую производится замена

### Strip filter

Фильтр strip просто удаляет начальные и конечные пробелы в каждой строке.

Параметры и их значения по умолчанию:

- `startswith: string = ""`
  : удалять только строки, начинающиеся с _startswith_ (игнорируя
  ведущие пробелы). Если пусто — удаляются все строки.

- `leading: bool = true`
  : удалять ведущие пробелы

- `trailing: bool = true`
  : удалять завершающие пробелы

### StdTmpl filter

Фильтр stdtmpl предоставляет простой шаблонизатор для Nim. Этот фильтр
использует построчный парсер: строки, начинающиеся с _meta character_
(по умолчанию `#`) содержат Nim-код, остальные строки вставляются как есть.
Поскольку основанный на отступах синтаксис не подходит для шаблонизатора,
операторы управления потоком требуют закрывающих `end X`-делимитеров.

Параметры и их значения по умолчанию:

- `metaChar: char = '#'`
  : префикс для строки, содержащей Nim-код

- `subsChar: char = '$'`
  : префикс для Nim-выражения внутри шаблонной строки

- `conc: string = " & "`
  : операция конкатенации

- `emit: string = "result.add"`
  : операция для добавления строкового литерала

- `toString: string = "$"`
  : операция, применяемая к каждому выражению для приведения к строке

Пример:

    #? stdtmpl | standard
    #proc generateHTMLPage(title, currentTab, content: string,
    #                      tabs: openArray[string]): string =
    #  result = ""
    <head><title>$title</title></head>
    <body>
      <div id="menu">
        <ul>
      #for tab in items(tabs):
        #if currentTab == tab:
        <li><a id="selected"
        #else:
        <li><a
        #end if
        href="${tab}.html">$tab</a></li>
      #end for
        </ul>
      </div>
      <div id="content">
        $content
        A dollar: $$.
      </div>
    </body>

Фильтр преобразует это в:

```nim
proc generateHTMLPage(title, currentTab, content: string,
                      tabs: openArray[string]): string =
  result = ""
  result.add("<head><title>" & $(title) & "</title></head>\n" &
    "<body>\n" &
    "  <div id=\"menu\">\n" &
    "    <ul>\n")
  for tab in items(tabs):
    if currentTab == tab:
      result.add("    <li><a id=\"selected\" \n")
    else:
      result.add("    <li><a\n")
    #end
    result.add("    href=\"" & $(tab) & ".html">" & $(tab) & "</a></li>\n")
  #end
  result.add("    </ul>\n" &
    "  </div>\n" &
    "  <div id=\"content\">\n" &
    "    " & $(content) & "\n" &
    "    A dollar: $.\n" &
    "  </div>\n" &
    "</body>\n")
```

Каждая строка, не начинающаяся с meta-символа (с учётом ведущих пробелов),
преобразуется в строковый литерал, который добавляется в `result`.

Символ подстановки вводит Nim-выражение _e_ в строковый литерал. _e_
преобразуется в строку с помощью операции _toString_, по умолчанию `$`. Для
строгой проверки типов установите `toString` в пустую строку. _e_ должен
соответствовать следующему PEG-паттерну:

    e <- [a-zA-Z\128-\255][a-zA-Z0-9\128-\255_.]* / '{' x '}'
    x <- '{' x+ '}' / [^}]*

Чтобы получить одиночный символ подстановки, его нужно удвоить: `$$`
даёт `$`.

Шаблонизатор достаточно гибок. Легко написать процедуру, которая будет
записывать сгенерированный код прямо в файл:

    #? stdtmpl(emit="f.write") | standard
    #proc writeHTMLPage(f: File, title, currentTab, content: string,
    #                   tabs: openArray[string]) =
    <head><title>$title</title></head>
    <body>
      <div id="menu">
        <ul>
      #for tab in items(tabs):
        #if currentTab == tab:
        <li><a id="selected"
        #else:
        <li><a
        #end if
        href="${tab}.html" title = "$title - $tab">$tab</a></li>
      #end for
        </ul>
      </div>
      <div id="content">
        $content
        A dollar: $$.
      </div>
    </body>

        <div id="content">
          $content
          A dollar: $$.
        </div>
      </body>

Фильтр преобразует это в:

```nim
proc generateHTMLPage(title, currentTab, content: string,
                      tabs: openArray[string]): string =
  result = ""
  result.add("<head><title>" & $(title) & "</title></head>\n" &
    "<body>\n" &
    "  <div id=\"menu\">\n" &
    "    <ul>\n")
  for tab in items(tabs):
    if currentTab == tab:
      result.add("    <li><a id=\"selected\" \n")
    else:
      result.add("    <li><a\n")
    #end
    result.add("    href=\"" & $(tab) & ".html">" & $(tab) & "</a></li>\n")
  #end
  result.add("    </ul>\n" &
    "  </div>\n" &
    "  <div id=\"content\">\n" &
    "    " & $(content) & "\n" &
    "    A dollar: $.\n" &
    "  </div>\n" &
    "</body>\n")
```

Каждая строка, не начинающаяся с meta-символа (с учётом ведущих пробелов),
преобразуется в строковый литерал, который добавляется в `result`.

Символ подстановки вводит Nim-выражение _e_ в строковый литерал. _e_
преобразуется в строку с помощью операции _toString_, по умолчанию `$`. Для
строгой проверки типов установите `toString` в пустую строку. _e_ должен
соответствовать следующему PEG-паттерну:

      e <- [a-zA-Z\128-\255][a-zA-Z0-9\128-\255_.]* / '{' x '}'
      x <- '{' x+ '}' / [^}]*

Чтобы получить одиночный символ подстановки, его нужно удвоить: `$$`
даёт `$`.

Шаблонизатор достаточно гибок. Легко написать процедуру, которая будет
записывать сгенерированный код прямо в файл:

      #? stdtmpl(emit="f.write") | standard
      #proc writeHTMLPage(f: File, title, currentTab, content: string,
      #                   tabs: openArray[string]) =
      <head><title>$title</title></head>
      <body>
        <div id="menu">
          <ul>
        #for tab in items(tabs):
          #if currentTab == tab:
          <li><a id="selected"
          #else:
          <li><a
          #end if
          href="${tab}.html" title = "$title - $tab">$tab</a></li>
        #end for
          </ul>
        </div>
        <div id="content">
          $content
          A dollar: $$.
        </div>
      </body>

```


```
