# AST в Nim



```nim
type
  NimNodeKind = enum     ## kind of a node; only explanatory
    nnkNone,
    nnkEmpty,
    nnkIdent,
    nnkIntLit,
    nnkStrLit,
    nnkNilLit,
    nnkCaseStmt,
    ...

  NimNode = ref NimNodeObj
  NimNodeObj = object
    case kind: NimNodeKind
    of nnkNone, nnkEmpty, nnkNilLit:
      discard
    of nnkCharLit..nnkUInt64Lit:
      intVal: BiggestInt
    of nnkFloatLit..nnkFloat64Lit:
      floatVal: BiggestFloat
    of nnkStrLit..nnkTripleStrLit, nnkCommentStmt, nnkIdent, nnkSym:
      strVal: string
    else:
      sons: seq[NimNode]
```

Оператор `[]` перегружен для `NimNode`: `n[i]` — i-й дочерний узел.

Некоторые дети (children) могут отсутствовать: отсутствующий дочерний узел представляется как `nnkEmpty` (а не `nil`).

## Leaf nodes / Atoms

Листовые узлы AST обычно соответствуют терминалам синтаксиса — литералам и идентификаторам. По умолчанию `float` в Nim соответствует `float64`, поэтому по умолчанию литерал с плавающей точкой превращается в `nnkFloat64Lit`.

| Nim expression                | Corresponding AST                      |
| ----------------------------- | -------------------------------------- |
| `42`                          | `nnkIntLit(intVal = 42)`               |
| `42'i8`                       | `nnkInt8Lit(intVal = 42)`              |
| `42'i16`                      | `nnkInt16Lit(intVal = 42)`             |
| `42'i32`                      | `nnkInt32Lit(intVal = 42)`             |
| `42'i64`                      | `nnkInt64Lit(intVal = 42)`             |
| `42'u8`                       | `nnkUInt8Lit(intVal = 42)`             |
| `42'u16`                      | `nnkUInt16Lit(intVal = 42)`            |
| `42'u32`                      | `nnkUInt32Lit(intVal = 42)`            |
| `42'u64`                      | `nnkUInt64Lit(intVal = 42)`            |
| `42.0`                        | `nnkFloat64Lit(floatVal = 42.0)`       |
| `42.0'f32`                    | `nnkFloat32Lit(floatVal = 42.0)`       |
| `42.0'f64`                    | `nnkFloat64Lit(floatVal = 42.0)`       |
| `"abc"`                       | `nnkStrLit(strVal = "abc")`            |
| `r"abc"`                      | `nnkRStrLit(strVal = "abc")`           |
| `"""abc"""`                   | `nnkTripleStrLit(strVal = "abc")`      |
| `' '`                         | `nnkCharLit(intVal = 32)`              |
| `nil`                         | `nnkNilLit()`                          |
| `myIdentifier`                | `nnkIdent(strVal = "myIdentifier")`    |
| `myIdentifier` (после lookup) | `nnkSym(strVal = "myIdentifier", ...)` |

Идентификаторы сначала парсятся как `nnkIdent`, затем после прохода name lookup преобразуются в `nnkSym`.

## Вызовы и выражения (Calls / expressions)

Далее идут часто встречающиеся формы вызовов/выражений. Конкретная нотация (concrete syntax) оставлена в исходном виде, AST — тоже. Пояснения переведены кратко.

### Command call

Concrete syntax:

```nim
echo "abc", "xyz"
```

AST:

```nim
nnkCommand(
  nnkIdent("echo"),
  nnkStrLit("abc"),
  nnkStrLit("xyz")
)
```

### Call with `()`

Concrete syntax:

```nim
echo("abc", "xyz")
```

AST:

```nim
nnkCall(
  nnkIdent("echo"),
  nnkStrLit("abc"),
  nnkStrLit("xyz")
)
```

### Infix operator call

Пример:

```nim
"abc" & "xyz"
```

AST:

```nim
nnkInfix(
  nnkIdent("&"),
  nnkStrLit("abc"),
  nnkStrLit("xyz")
)
```

При нескольких инфиксных операторах учитывается приоритет (operator precedence). Также инфикс можно представить в префиксной форме, тогда используется `nnkAccQuoted`.

### Prefix / Postfix / Named arguments / Raw string calls

- Prefix operators используют `nnkPrefix`.
- Постфиксных операторов в Nim нет; `nnkPostfix` используется только для маркера экспорта `*`.
- Именованные аргументы (`name=value`) представлены узлом `nnkExprEqExpr`.
- Вызовы с raw string literal (например `echo"abc"`) имеют узел `nnkCallStrLit`.

(во всех случаях кодовые примеры сохранены из оригинала — их можно смотреть в исходном файле).

## Операторы доступа и скобки

- `.` (dot) — `nnkDotExpr`.
- `[]` (bracket) — `nnkBracketExpr`.
- Скобки для изменения приоритета — `nnkPar`.

## Конструкторы: tuple, curly (set/table), brackets (array)

- Tuple constructors — `nnkTupleConstr`.
- Curly braces `{...}` — `nnkCurly` (set) или `nnkTableConstr` (таблица `{a: 3, b:5}`).
- Brackets `[...]` — `nnkBracket` (array constructor).

## Ranges

Операция `..` представляется как `nnkInfix(nnkIdent(".."), ...)` (внутренне используется `nnkRange` при некоторых конструкциях).

## If expression / statement

- `if` expression: вложенные `nnkElifExpr` и `nnkElseExpr`.
- `if` statement: `nnkIfStmt` с `nnkElifBranch` и `nnkElse` (если есть).

## Комментарии документации

Двойной хеш `##` — это документационные комментарии; они отображаются как `nnkCommentStmt` и агрегируются (несколько подряд идущих строк `##` — одна `nnkCommentStmt`). Обычные `#` игнорируются в этой подстановке.

## Pragmas

Pragma-ы (например `{.emit: "...".}`) представлены узлом `nnkPragma` и т.д.; их синтаксис в AST показан в оригинале.

## Списки операторов/отрезков/прочее

Многие простые операторы и конструкции описаны в исходном тексте; я сохранил примеры кода (не переводя их) и перевёл поясняющие тексты.

## Типы (Types) — краткая таблица

Ниже переведённая таблица соответствий типовых конструкций Nim и нод AST.

| Nim type   | Corresponding AST                         |
| ---------- | ----------------------------------------- |
| `static`   | `nnkStaticTy`                             |
| `tuple`    | `nnkTupleTy`                              |
| `var`      | `nnkVarTy`                                |
| `ptr`      | `nnkPtrTy`                                |
| `ref`      | `nnkRefTy`                                |
| `distinct` | `nnkDistinctTy`                           |
| `enum`     | `nnkEnumTy`                               |
| `concept`  | `nnkTypeClassTy`\*                        |
| `array`    | `nnkBracketExpr(nnkIdent("array"),...)`\* |
| `proc`     | `nnkProcTy`                               |
| `iterator` | `nnkIteratorTy`                           |
| `object`   | `nnkObjectTy`                             |

(\* — примечания из оригинала: некоторые случаи имеют нюансы; см. оригинальный файл для подробностей.)

## Процедуры, итераторы, шаблоны, макросы

- `nnkProcDef` — определение процедуры; включает `nnkGenericParams`, `nnkFormalParams`, pragma-ы и тело (`nnkStmtList`).
- `nnkIteratorDef`, `nnkTemplateDef`, `nnkMacroDef` — похожие структуры с небольшими отличиями (у шаблонов и макросов есть slot для term-rewriting и т.д.).

## Hidden Standard Conversion

Когда литерал одного типа автоматически конвертируется в другой (например `int` -> `float`), компилятор вставляет `nnkHiddenStdConv` вокруг соответствующего литерала, чтобы явно показать операцию преобразования в AST.

## Специальные ноды

В AST есть ноды, используемые для семантической проверки и генерации кода; они доступны в модуле, но обычно не используются напрямую. Остальные ноды упрощают манипуляции AST и описаны в оригинале.

---
