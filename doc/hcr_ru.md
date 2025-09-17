# Hot code reloading
.. default-role:: code
.. include:: rstcommon.rst

Опция `hotCodeReloading` включает специальный режим компиляции, в котором
изменения в коде могут быть автоматически применены к уже запущенной
программе. Перезагрузка выполняется на уровне отдельного модуля. Когда модуль
перезагружается, любые добавленные глобальные переменные будут
инициализированы, тогда как остальной верхнеуровневый код модуля не будет
повторно выполняться, и состояние существующих глобальных переменных
сохранится.

## Basic workflow

В настоящий момент горячая перезагрузка кода не работает для самого
главного модуля (main). Поэтому требуется вспомогательный модуль, в котором
будет находиться основная логика, которую мы хотим менять во время
разработки.

В этом примере мы используем SDL2 для создания окна и перезагружаем
логику при нажатии `F9`. Важные строки помечены `#***`. Для установки
SDL2 можно воспользоваться `nimble install sdl2`:cmd:.

```nim
# logic.nim
import sdl2

#*** import the hotcodereloading stdlib module ***
import std/hotcodereloading

var runGame*: bool = true
var window: WindowPtr
var renderer: RendererPtr
var evt = sdl2.defaultEvent

proc init*() =
  discard sdl2.init(INIT_EVERYTHING)
  window = createWindow("testing", SDL_WINDOWPOS_UNDEFINED.cint, SDL_WINDOWPOS_UNDEFINED.cint, 640, 480, 0'u32)
  assert(window != nil, $sdl2.getError())
  renderer = createRenderer(window, -1, RENDERER_SOFTWARE)
  assert(renderer != nil, $sdl2.getError())

proc destroy*() =
  destroyRenderer(renderer)
  destroyWindow(window)

var posX: cint = 1
var posY: cint = 0
var dX: cint = 1
var dY: cint = 1

proc update*() =
  while pollEvent(evt):
    if evt.kind == QuitEvent:
      runGame = false
      break
    if evt.kind == KeyDown:
      if evt.key.keysym.scancode == SDL_SCANCODE_ESCAPE: runGame = false
      elif evt.key.keysym.scancode == SDL_SCANCODE_F9:
        #*** reload this logic.nim module on the F9 keypress ***
        performCodeReload()

  # draw a bouncing rectangle:
  posX += dX
  posY += dY

  if posX >= 640: dX = -2
  if posX <= 0: dX = +2
  if posY >= 480: dY = -2
  if posY <= 0: dY = +2

  discard renderer.setDrawColor(0, 0, 255, 255)
  discard renderer.clear()
  discard renderer.setDrawColor(255, 128, 128, 0)

  var rect: Rect = (x: posX - 25, y: posY - 25, w: 50.cint, h: 50.cint)
  discard renderer.fillRect(rect)
  delay(16)
  renderer.present()
```


```nim
# mymain.nim
import logic

proc main() =
  init()
  while runGame:
    update()
  destroy()

main()
```

Скомпилируйте пример так:

```cmd
nim c --hotcodereloading:on mymain.nim
```

Запустите программу и держите её работающей:

```cmd
# Unix:
mymain &
# or Windows (click on the .exe)
mymain.exe
# edit
```

Например, измените строку

```nim
discard renderer.setDrawColor(255, 128, 128, 0)
```

на

```nim
discard renderer.setDrawColor(255, 255, 128, 0)
```

(это изменит цвет прямоугольника). Затем перекомпилируйте проект, но не
перезапускайте и не завершайте `mymain.exe`:

```cmd
nim c --hotcodereloading:on mymain.nim
```

Теперь сфокусируйте SDL-окно `mymain`, нажмите `F9` и наблюдайте за
обновлённой версией программы.

## Reloading API

Можно использовать специальные обработчики событий `beforeCodeReload` и
`afterCodeReload`, чтобы сбрасывать состояние каких-то переменных или
заставлять выполняться определённые операторы:

```nim
var
  settings = initTable[string, string]()
  lastReload: Time

for k, v in loadSettings():
  settings[k] = v

initProgram()

afterCodeReload:
  lastReload = now()
  resetProgramState()
```

При каждой перезагрузке сначала будут выполнены все `beforeCodeReload`
обработчики, зарегистрированные в предыдущей версии программы, а затем
все `afterCodeReload` обработчики из загруженного кода. Обратите внимание,
что обработчики из модулей, которые не были перезагружены, также будут
выполнены; чтобы этого избежать, можно проверять изменение модуля с помощью
`hasModuleChanged()`:

```nim
import mydb

var myCache = initTable[Key, Value]()

afterCodeReload:
  if hasModuleChanged(mydb):
    resetCache(myCache)
```

Механизм горячей перезагрузки основан на подмене динамических библиотек для
нативных целей и на прямом манипулировании глобальным пространством имён
для JavaScript. Компилятор Nim не определяет сам механизм обнаружения
момента, когда необходимо выполнить перезагрузку — ожидается, что код
программы сам будет вызывать `performCodeReload()` всякий раз, когда хочет
перезагрузить код.

Ожидается, что большинство проектов реализуют триггер перезагрузки через
систему сборки и IPC-нотификации, но также возможен polling с использованием
`hasAnyModuleChanged()` API.

Чтобы получить доступ к `beforeCodeReload`, `afterCodeReload`,
`hasModuleChanged` или `hasAnyModuleChanged`, необходимо импортировать модуль
`hotcodereloading`.

## Native code targets

Нативные проекты с включённой горячей перезагрузкой будут неявно собраны с
опцией `-d:useNimRtl` и будут зависеть от библиотек `nimrtl` и `nimhcr`,
реализующих рантайм горячей перезагрузки. Обе библиотеки находятся в папке
`lib` Nim и могут быть скомпилированы в динамические библиотеки для
исполнения примера выше. Пример компиляции `nimhcr.nim` и `nimrtl.nim` при
установленном choosenim:

```console
# Unix/MacOS
# Make sure you are in the directory containing your .nim files
$ cd your-source-directory

# Compile two required files and set their output directory to current dir
$ nim c --outdir:$PWD ~/.choosenim/toolchains/nim-#devel/lib/nimhcr.nim
$ nim c --outdir:$PWD ~/.choosenim/toolchains/nim-#devel/lib/nimrtl.nim

# verify that you have two files named libnimhcr and libnimrtl in your
# source directory (.dll for Windows, .so for Unix, .dylib for MacOS)
```

Все модули проекта будут скомпилированы в отдельные динамические библиотеки
и помещены в каталог `nimcache`. Обратите внимание, что во время выполнения
рантайм перезагрузки будет загружать копии этих библиотек, чтобы не мешать
новым командам сборки.

Главный модуль программы считается нередактируемым (non-reloadable). Обратите
внимание, что процедуры из перезагружаемых модулей не должны находиться в
стеке вызовов в момент вызова `performCodeReload`. Поэтому главный модуль —
подходящее место для реализации цикла программы, который может вызывать
`performCodeReload`.

Обратите внимание: перезагрузка будет невозможна, если была изменена любая
из определений типов в программе. Если используются closure-итераторы
(напрямую или через async-код), — перезагружённые определения будут
влиять только на вновь создаваемые экземпляры. Существующие экземпляры
итераторов будут исполнять свой оригинальный код до завершения.

## JavaScript target

После компиляции кода с поддержкой горячей перезагрузки удобным решением для
реализации самой перезагрузки в браузере является использование фреймворка
например [LiveReload](https://livereload.com/) или [BrowserSync](https://browsersync.io/).
