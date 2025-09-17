# Embedded Stack Trace Profiler (ESTP) — Руководство пользователя

.. default-role:: code
.. include:: rstcommon.rst

:Author: Andreas Rumpf
:Version: |nimversion|

Nim поставляется с платформонезависимым профайлером — Embedded Stack Trace
Profiler (ESTP). Профайлер встраивается в исполняемый файл. Чтобы
активировать профайлер, выполните:

- скомпилируйте программу с опциями командной строки `--profiler:on --stackTrace:on`;
- импортируйте модуль `nimprof`;
- запустите программу как обычно.

Вы также можете изучить исходный код `nimprof`, чтобы понять, как
реализовать собственный профайлер.

Переключатель `--profiler:on` определяет условный символ `profiler`.
Можно использовать `when compileOption("profiler")` чтобы сделать переключение
бесшовным. Если `profiler` равен `off`, программа выполняется как обычно,
иначе выполняется профилирование.

```nim
when compileOption("profiler"):
  import std/nimprof
```

После завершения программы профайлер создаст файл `profile_results.txt`,
содержащий результаты профилирования.

Поскольку профайлер работает путём исследования стектрейсов, важно, чтобы
опция `--stackTrace:on` была активирована. К сожалению, это делает
профилирующую сборку значительно медленнее, чем релизную.

## Memory profiler

ESTP можно использовать и как профайлер памяти — чтобы увидеть, какие
стектрейсы выделяют больше всего памяти и создают наибольшее давление на GC.
Он также помогает находить утечки памяти. Для включения memory profiler
выполните:

- скомпилируйте программу с опциями
  `--profiler:off --stackTrace:on -d:memProfiler` (да, используется `--profiler:off`);
- импортируйте модуль `nimprof`;
- запустите программу.

Определите символ `ignoreAllocationSize`, если хотите учитывать только число
алокаций, игнорируя их размеры.

## Пример файла результатов

Файл результатов перечисляет стектрейсы, упорядоченные по значимости.

Ниже приведён пример, сгенерированный профилированием самого компилятора
Nim: видно, что в сумме 5.4% времени выполнения было потрачено в
`crcFromRope` или его потомках.

В общем случае стектрейсы позволяют быстро локализовать проблему — трасса
работает как объяснение; в традиционных профайлерах легко найти дорогие
leaf-функции, но понять причину их частого вызова бывает сложно.

Пример вывода (усечённый):

```
total executions of each stack trace:
Entry: 0/3391 Calls: 84/4160 = 2.0% [sum: 84; 84/4160 = 2.0%]
  newCrcFromRopeAux
  crcFromRope
  writeRopeIfNotEqual
  shouldRecompile
  writeModule
  myClose
  closePasses
  processModule
  CompileModule
  CompileProject
  CommandCompileToC
  MainCommand
  HandleCmdLine
  nim
Entry: 1/3391 Calls: 46/4160 = 1.1% [sum: 130; 130/4160 = 3.1%]
  updateCrc32
  newCrcFromRopeAux
  crcFromRope
  writeRopeIfNotEqual
  shouldRecompile
  writeModule
  myClose
  closePasses
  processModule
  CompileModule
  CompileProject
  CommandCompileToC
  MainCommand
  HandleCmdLine
  nim
Entry: 2/3391 Calls: 41/4160 = 0.99% [sum: 171; 171/4160 = 4.1%]
  updateCrc32
  updateCrc32
  newCrcFromRopeAux
  crcFromRope
  writeRopeIfNotEqual
  shouldRecompile
  writeModule
  myClose
  closePasses
  processModule
  CompileModule
  CompileProject
  CommandCompileToC
  MainCommand
  HandleCmdLine
  nim
...
```
