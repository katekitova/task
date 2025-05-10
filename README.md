# Интерпретатор модельного языка программирования
*Выполнила* **Китова Екатерина Денисовна**, 215 группа

---
## Описание

Требовалось разработать и реализовать интерпретатор модельного языка программирования на С++, с поддержкой:
* **Объявлений и типов**: `int`, `real`, `string`
* **Управляющих конструкций**:
  * Условный оператор `if (cond) stmt else stmt`
  * Циклы `while (cond) stmt` и `for (init; cond; step) stmt`
  * Оператор `continue` внутри циклов
  * Многозначный `case (expr) of` с диапазонами и дефолт-веткой `else`
* **Ввод/вывод**: `read(var)`, `write(expr, ...)`
* **Выражения**: арифметические, логические, операции сравнения, строковая конкатенация

Язык компилируется в ПОЛИЗ — внутреннее представление `vector<Instruction>`, а затем выполняется стековым интерпретатором.

## Структура проекта
### `interpreter.cpp`
1. **Lexer** (`class Lexer`)

   * Складывает исходный текст в последовательность токенов (`Token`): идентификаторы, числа, строки, ключевые слова, символы.
   * Пропускает пробелы и комментарии `/* ... */`.

2. **Parser** (`class Parser`)

   * Разбирает программу, опираясь на рекурсивный спуск:

     * Декларации переменных и их инициализация
     * Инструкции (`parseStatement`): блоки `{}`, `if`, `while`, `for`, `case`, `read`, `write`, `continue`
     * Выражения с поддержкой приоритетов: `parseExpression`, `parseAdditive`, `parseTerm`, `parseFactor`, `parsePrimary`
   * Генерирует инструкции ПОЛИЗ (`emit`, `emitReal`, `backpatch`) для каждой конструкции:

     * Условные и безусловные переходы (`OP_JMP_FALSE`, `OP_JMP`)
     * Арифметика и операции сравнения/логики
     * Управление циклом и `case`

3. **Полиз-Интерпретатор** (`class Interpreter`)

   * Интерпретирует `vector<Instruction> code` и работает со стеком `vector<Value> stack`.
   * Для каждой операции (`switch` по `inst.op`) выполняет:

     * Манипуляции со стеком: `push`, `pop`
     * Арифметику, сравнения, логические операции
     * Чтение/запись через `cin`/`cout`
     * Переходы управления (`pc = operand`)

4. **`main`**

   * Загружает исходник из файла или STDIN
   * Запускает лексер, парсер, затем интерпретатор

## Сборка

```bash
# Скомпилировать
g++ -std=c++17 -Wall -O2 interpreter.cpp -o interpreter

# Запуск c файлом
./interpreter tests/program.txt
```

или

```bash
# Запуск без файла(для конца ввода Control+D)
./interpreter 
```

Где `interpreter.cpp` содержит реализацию Lexer, Parser и Interpreter, а `tests/program.txt` — текст программы на мини-языке.

## Пример программы

```pascal
program
{
  int a = 51, b = 6, c;
  string x = "abc", y, z = "abcd";

  c = (a + b) * 2;
  if (c >= 100 or x == z)
  {
    read(y);
    write(y);
    write(x + y + z, c);
  }
  else
    c = a = 21;
  while (c > 100)
  {
    c = c - 5;
    write(c);
    x = x + "step";
  }
  write(x);
}
```
**Входные данные:**
```
Введите значение для переменной y: 1234
```

**Ожидаемый вывод:**

```
"1234" 
"abc1234abcd" 114 
109 
104 
99 
"abcstepstepstep"
```

# Отчёт
## 1. Объектная модель интерпретатора

- **`Token`**
  - `TokenType type`
  - `string text`
  - `int intValue`
  - `double realValue`
  - `int line`

- **`Lexer`**
  - Поля: `string src`, `size_t pos`, `int line`
  - Методы: `getNextToken()`, `skipWhitespaceAndComments()`

- **`Instruction`**
  - `Opcode op`
  - `int operand`
  - `double realOperand`
  - `string strOperand`

- **`Value`**
  - `ValueType type`
  - `int intVal`
  - `double realVal`
  - `string strVal`

- **`Interpreter`**
  - Поля:
    - `vector<Instruction> code`
    - `vector<pair<string,ValueType>> symTable`
    - `vector<Value> variables`
    - `vector<Value> stack`
    - `size_t pc`
  - Методы: `run()`, `push()`, `pop()`

- **`Parser`**
  - Поля:
    - `Lexer &lexer`
    - `Token curToken, buffer`
    - `bool hasBuffer`
    - `vector<Instruction> code`
    - `vector<pair<string,ValueType>> symbolTable`
    - `map<string,int> varIndex`
    - стеки для `continue`
  - Методы:
    - `parseProgram()`, `parseDeclarations()`, `parseStatement()`, `parseExpression()`
    - `parseIf()`, `parseWhile()`, `parseFor()`, `parseCase()`, `parseRead()`, `parseWrite()`, `parseContinue()`
    - Утилиты: `nextToken()`, `putBack()`, `expect()`, `error()`, `emit()`, `emitReal()`, `backpatch()`

---

## 2. Синтаксис реализуемого модельного языка

```ebnf
<program>      ::= "program" "{" <declarations> <statements> "}"
<declarations> ::= { <type> <ident> [ "=" <expr> ] { "," <ident> [ "=" <expr> ] } ";" }
<type>         ::= "int" | "real" | "string"

<statements>   ::= { <statement> }
<statement>    ::= <block>
                 | "if" "(" <expr> ")" <statement> [ "else" <statement> ]
                 | "while" "(" <expr> ")" <statement>
                 | "for" "(" <assign_stmt> ";" <expr> ";" <expr> ")" <statement>
                 | "case" "(" <expr> ")" "of" { <literal> ":" <statement> } "end" ";"
                 | "read" "(" <ident> ")" ";"
                 | "write" "(" <expr> { "," <expr> } ")" ";"
                 | "continue" ";"
                 | <assign_stmt> ";"

<block>        ::= "{" <statements> "}"
<assign_stmt>  ::= <ident> "=" <expr>

<expr>         ::= <logic_or>
<logic_or>     ::= <logic_and> { "or" <logic_and> }
<logic_and>    ::= <equality> { "and" <equality> }
<equality>     ::= <relational> { ("==" | "!=") <relational> }
<relational>   ::= <additive> { ("<" | ">" | "<=" | ">=") <additive> }
<additive>     ::= <term> { ("+" | "-") <term> }  
<term>         ::= <factor> { ("*" | "/" | "%") <factor> }
<factor>       ::= [ "-" | "+" | "not" ] <primary>
<primary>      ::= <number> | <real> | <string_literal> | <ident> | "(" <expr> ")"

<literal>      ::= <number> | <string_literal>
```

---

## 3. Неопределённые в описании детали языка

1. **Идентификаторы**  
   - Регулярное выражение: `[A-Za-z_][A-Za-z0-9_]*`  
   - Максимальная длина: 128 символов

2. **Числовые константы**  
   - Целые: до 9 цифр (в пределах `int`)  
   - Вещественные: с одной точкой, минимум одна цифра до или после

3. **Строковые литералы**  
   - Ограждены двойными кавычками `"..."` 
   - Максимальная длина: 256 символов

4. **Комментарии**  
   - Блочные: `/* ... */`, без вложенности, обязательное закрытие

5. **Форматы ввода/вывода**  
   - `read(var)`: считывает одну строку до `
`  
   - `write(...)`: элементы через пробел, строки в кавычках

---

## 4. Конечный автомат лексического анализатора

| Состояние      | Входной символ      | Переход в     | Действие                            |
| -------------- | ------------------- | ------------- | ----------------------------------- |
| **S_0 (start)**| `\s`, ``            | S_0           | `pos++`, при `` → `line++`          |
|                | `/`                 | S_1           | `pos++`                             |
|                | `*`                 | S_0(символ)   | токен `*`                           |
|                | `"`                 | S_STR         | начало строкового литерала          |
|                | буква или `_`       | S_ID          | сбор идентификатора/ключевого слова |
|                | цифра               | S_NUM         | сбор числа (возможно реал)          |
|                | `<`, `>`, `=`, `!`  | S_CMP         | проверка `=` для `<=,>=,==,!=`      |
|                | др. символы         | S_0           | одиночный символ                    |
| **S_1**         | `*`                | S_COM         | начало комментария                  |
|                | иначе               | S_0           | возврат `/`                         |
| **S_COM**      | `*`                 | S_COM_END?    |                                     |
|                | иначе               | S_COM         |                                     |
| **S_COM_END?** | `/`                 | S_0           | выход из комментария                |
|                | иначе               | S_COM         |                                     |
| **S_STR**      | `"`                 | S_0           | закрыть строку, токен               |
|                | EOF                 | —             | ошибка: не закрыта строка           |
|                | иначе               | S_STR         | накопление символов                 |
| **S_ID**       | буква/цифра/`_`     | S_ID          |                                     |
|                | иначе               | S_0           | возврат токена; `putBack`           |
| **S_NUM**      | цифра               | S_NUM         |                                     |
|                | `.`                 | S_NUM_REAL    | пометка вещественного               |
|                | иначе               | S_0           | возврат токена; `putBack`           |
| **S_NUM_REAL** | цифра               | S_NUM_REAL    |                                     |
|                | иначе               | S_0           | возврат токена; `putBack`           |

---

## 5. Обоснование применимости метода рекурсивного спуска

1. **LL(1)-грамматика**  
   - Без левой рекурсии, решение по первому токену.

2. **Модульность**  
   - Каждый нетерминал — отдельный метод `parseXXX()`, где `XXX` это имя нетерминала.

3. **Backpatch**  
   - `emit()` возвращает индекс, упрощая дозаполнение переходов.

---

## 6. Комплекты A- и B-тестов

### A-Тесты
- `programm.txt` – пример из задания  
- `if_arf.txt` – оператор `if` и арифметика  
- `while_str.txt` – цикл `while` и строковые конструкции  
- `real.txt` – работа с типом `real`  
- `case.txt` – `case ... of ... end`  
- `for_cont.txt` – цикл `for(...;...;...)` и `continue` 
- `unary.txt` – унарные `+`, `-`, `not`  
- `lazy.txt` – ленивые вычисления в логических выражениях
- `test_poliz.txt` - вывод ПОЛИЗа

### B-Тесты
- `bad.txt` – использование неописанной переменной  
- `zero.txt` – деление на ноль  
- `err_test1.txt` – незакрытый комментарий  
- `err_test.txt` – незакрытая строковая константа  

---

## 7. Вывод ПОЛИЗа на экран(дополнительное задание)
**Описание функций ПОЛИЗ**

- `PUSH_INT`  — кладёт целочисленную константу n в стек.
- `PUSH_STRING`  — кладёт строковую константу s в стек.
- `STORE`  — берёт значение из стека и сохраняет его в переменной с индексом i.
- `LOAD`  — загружает значение переменной с индексом i в стек.
- `ADD`— берёт два значения из стека, выполняет сложение (для строк — конкатенацию), результат кладёт в стек.
- `SUB` — вычитание: берёт два числа из стека, вычитает второе из первого, результат в стек.
- `MUL` — умножение: два числа → их произведение.
- `DIV` —  деление: два числа -> частное.
- `EQ` — проверка равенства: снимает два значения, сравнивает, кладёт true(1)/false(0).
- `GE` — операция >=: сравнение двух чисел → true/false.
- `GT` — операция >: сравнение двух чисел → true/false.
- `JMP`  — безусловный переход на метку (номер инструкции) m.
- `JMP_FALSE`  — условный переход: снимает флаг (0/1) из стека, если false (0) — переходит на m, иначе продолжает.
- `READ`  — читает значение из ввода (например, read(y)) и сохраняет в переменной i.
- `WRITE`  — берет из стека k значений (в обратном порядке кладилась), выводит их на экран.

Также есть аналогичные PUSH_REAL, NE, AND, OR, NOT


**Ниже представлен разбор программы из условий задания**
```pascal
program
{
  int a = 51, b = 6, c;
  string x = "abc", y, z = "abcd";

  c = (a + b) * 2;
  if (c >= 100 or x == z)
  {
    read(y);
    write(y);
    write(x + y + z, c);
  }
  else
    c = a = 21;
  while (c > 100)
  {
    c = c - 5;
    write(c);
    x = x + "step";
  }
  write(x);
}
```
**ПОЛИЗ:**

```
=== ПОЛИЗ ===
0: PUSH_INT 51
1: STORE 0
2: PUSH_INT 6
3: STORE 1
4: PUSH_STRING "abc"
5: STORE 3
6: PUSH_STRING "abcd"
7: STORE 5
8: LOAD 0
9: LOAD 1
10: ADD
11: PUSH_INT 2
12: MUL
13: STORE 2
14: LOAD 2
15: PUSH_INT 100
16: GE
17: JMP_FALSE 20
18: PUSH_INT 1
19: JMP 23
20: LOAD 3
21: LOAD 5
22: EQ
23: JMP_FALSE 35
24: READ 4
25: LOAD 4
26: WRITE 1
27: LOAD 3
28: LOAD 4
29: ADD
30: LOAD 5
31: ADD
32: LOAD 2
33: WRITE 2
34: JMP 38
35: PUSH_INT 21
36: STORE 0
37: STORE 2
38: LOAD 2
39: PUSH_INT 100
40: GT
41: JMP_FALSE 53
42: LOAD 2
43: PUSH_INT 5
44: SUB
45: STORE 2
46: LOAD 2
47: WRITE 1
48: LOAD 3
49: PUSH_STRING "step"
50: ADD
51: STORE 3
52: JMP 38
53: LOAD 3
54: WRITE 1
==============
```
### Пояснения
**Инициализация переменных**
```
0:  PUSH_INT 51       // кладём константу 51 в стек  
1:  STORE 0           // a = 51      
2:  PUSH_INT 6        // кладём константу 6  
3:  STORE 1           // b = 6      
4:  PUSH_STRING "abc" // кладём строку "abc"  
5:  STORE 3           // x = "abc"  
6:  PUSH_STRING "abcd"
7:  STORE 5           // z = "abcd"  
```

**Вычисление c = (a + b) * 2**
```
8:  LOAD 0    // кладём a  
9:  LOAD 1    // кладём b  
10: ADD      // a + b  
11: PUSH_INT 2  
12: MUL      // (a + b) * 2  
13: STORE 2  // c = (a + b) * 2  
```

**Условие if (c >= 100 or x == z)**
```
14: LOAD 2         // кладём c  
15: PUSH_INT 100  
16: GE             // проверяем c >= 100 -> true/false  
17: JMP_FALSE 20   // если false -> метка 20  
18: PUSH_INT 1     // (true)  
19: JMP 23         // пропустить проверку x==z  
20: LOAD 3         // кладём x  
21: LOAD 5         // кладём z  
22: EQ             // проверяем x == z  
23: JMP_FALSE 35   // если false -> метка 35 (начало else)  
```

**Ветка then**
```
24: READ 4         // read(y) -> y в ячейку 4  
25: LOAD 4         // кладём y  
26: WRITE 1        // write(y)  
27: LOAD 3         // кладём x  
28: LOAD 4         // кладём y  
29: ADD            // x + y  
30: LOAD 5         // кладём z  
31: ADD            // (x + y) + z  
32: LOAD 2         // кладём c  
33: WRITE 2        // write(x+y+z, c)  
34: JMP 38         // перейти к циклу, пропускаем else  
```

**Ветка else**
```
35: PUSH_INT 21  
36: STORE 0        // a = 21  
37: STORE 2        // c = a = 21  
```

**Цикл while (c > 100)**
```
38: LOAD 2        // кладём c  
39: PUSH_INT 100  
40: GT            // проверяем c > 100  
41: JMP_FALSE 53  // если false -> метка 53 (выход из цикла)  

42: LOAD 2        // тело цикла: кладём c  
43: PUSH_INT 5  
44: SUB           // c - 5  
45: STORE 2       // сохраняем новое c  
46: LOAD 2  
47: WRITE 1       // выводим c  
48: LOAD 3  
49: PUSH_STRING "step"
50: ADD           // x = x + "step"  
51: STORE 3       // сохраняем x  
52: JMP 38        // повторная проверка  
```

**Финальный вывод write(x)**
```
53: LOAD 3    // кладём x  
54: WRITE 1   // вывод x 
```
