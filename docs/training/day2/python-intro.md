# Python for the HCPI Platform

<!-- **Duration:** ~1 hour, deliberately slow.
**Audience:** Developers who already write code in some other language (Java, C#, PHP, JavaScript, Go, …) and need just enough Python to be productive inside HCPI.
**Outcome:** You'll recognise every Python idiom HCPI uses and be able to read any model file without getting stuck on syntax. -->

By the end of this page you'll be able to read any Python file in HCPI and follow what it's doing. We move slowly, build everything in small files, and write code rather than just talk about it.

## What Python is

Python is a programming language. A Python **program** is a plain text file (ending in `.py`) containing instructions you want the computer to follow. You **run** the file by giving it to the Python **interpreter**, which reads it top-to-bottom and does what it says.

That's the whole loop you'll repeat dozens of times today:

```
write some lines into a file  →  save it  →  run it  →  read what it printed  →  edit it again
```

HCPI is written in Python, so every back-end file you'll touch is one of these `.py` files. The same skills that work on a 10-line practice file work on a 10,000-line HCPI module.

## Setting up where you'll write code

Everything in this page happens **outside** HCPI, in a throwaway folder so you can't accidentally break your install.

### Open WSL in VS Code

If you're on Windows, open a WSL terminal (the same Ubuntu shell you used during installation) and create a practice folder:

```bash
mkdir -p ~/python-practice
cd ~/python-practice
code .
```

The last command, `code .`, opens VS Code in WSL pointing at the current folder. The first time, VS Code will install a small helper into WSL and ask you to install the **Remote – WSL** extension if you don't have it. Accept.

You'll know VS Code is in WSL mode when the bottom-left corner reads **WSL: Ubuntu** (or similar). Files you create here live in the WSL filesystem at `/home/<your-user>/python-practice/` and can be edited from VS Code as if they were native.

If you're on Linux, the same `code .` works directly.

### Open the integrated terminal

Inside VS Code, hit **Ctrl + `** (the backtick key, top-left of most keyboards) to open the integrated terminal. It opens in your current folder, in WSL. This is where you'll run your Python files.

### Verify Python is installed

In that terminal:

```bash
python3 --version
```

You should see something like `Python 3.11.x`. If not, install it — Day 1's [Installing Dependencies](../day1/dependencies.md) covers this.

### Your first file

In VS Code's file panel, create a new file called `hello.py` (Ctrl + N, then Ctrl + S to save with a name). Type one line:

```python
print("hello")
```

Save with **Ctrl + S**. Then in the terminal:

```bash
python3 hello.py
```

You should see `hello` print out. That's the loop. We'll repeat it.

!!! tip "Always save before running"
    Running a file runs the *saved* version on disk, not whatever's currently on screen in the editor. Ctrl+S is muscle memory you'll want to develop.

## 1. The `print` function and comments

`print(...)` displays text in the terminal. It's how you make a program "say something" you can see. Edit `hello.py` to:

```python
print("Hello, HCPI.")
print("This is line two.")
# This line is a comment — Python ignores it.
print("This is line three.")  # comments can also sit at the end of a line
```

Save and run again:

```
Hello, HCPI.
This is line two.
This is line three.
```

Two things to notice:

- **Lines run top to bottom.** Python reads line 1, then line 2, then line 3.
- **`#` starts a comment.** Everything from `#` to the end of the line is ignored when the program runs. Use comments to leave notes for yourself or other readers.

Whitespace around `print` matters — see the indentation in the next section.

## 2. Variables

A **variable** is a name that stands for a value. You create a variable by writing a name, an equals sign, and the value:

```python
greeting = "Hello"
year = 2026
pi = 3.14159
```

Now any time you write `greeting` later in the file, Python substitutes `"Hello"`. Create `variables.py`:

```python
greeting = "Hello"
name = "Alice"

print(greeting)
print(name)
print(greeting, name)   # print can take multiple values separated by commas
```

Run it:

```
Hello
Alice
Hello Alice
```

### Re-assigning

A variable doesn't keep its first value forever. Assigning again replaces it:

```python
count = 1
print(count)        # 1
count = count + 1
print(count)        # 2
count = "now I'm a string"
print(count)        # now I'm a string
```

In some languages a variable's type is fixed when you declare it. Not in Python — a variable just holds whatever value you last assigned. You **shouldn't** flip between types like the example above in real code (it confuses readers), but Python won't stop you.

### Naming rules

A variable name:

- Can contain letters, digits, and underscores (`_`).
- **Cannot start with a digit.** `total2 = 5` is fine; `2total = 5` is an error.
- Is case-sensitive: `total` and `Total` are different variables.
- Should be lowercase with underscores between words (`outlet_count`, not `outletCount` or `OutletCount`). This is the Python convention used everywhere in HCPI.
- Cannot be a Python reserved word: `if`, `for`, `class`, etc.

### Exercise 1

Create `intro.py` with three variables: your name, your age, the current year. Print a line that uses all three. (Don't worry yet about combining them into one nice sentence — just print them comma-separated.)

## 3. The basic types

Python tracks what kind of value each variable holds. The most common types you'll meet:

| Type | What it is | Example |
|---|---|---|
| `int` | Whole number | `x = 5` |
| `float` | Decimal number | `y = 3.14` |
| `str` | Text (a "string") | `name = "Alice"` |
| `bool` | True or false | `flag = True` |
| `NoneType` | "No value" / "missing" | `nothing = None` |

A few things that surprise newcomers:

**1. No separate "double" or "long".** Python has just one integer type (`int`) and one decimal type (`float`). Don't worry about the difference between them most of the time.

**2. Booleans are capitalised: `True` and `False`.** Not `true`/`TRUE`/`false`.

**3. `None` is special.** It's Python's word for "nothing here yet" or "no value." Different languages call this `null`, `nil`, or `undefined`. It is itself a value — `nothing = None` is a real assignment.

You can ask Python what type something is:

```python
x = 5
print(type(x))          # <class 'int'>
print(type(3.14))       # <class 'float'>
print(type("hi"))       # <class 'str'>
print(type(True))       # <class 'bool'>
print(type(None))       # <class 'NoneType'>
```

You won't write `type(...)` often in real code — it's just a way to check what Python thinks something is when you're debugging.

## 4. Numbers and math

```python
a = 10
b = 3

print(a + b)        # 13
print(a - b)        # 7
print(a * b)        # 30
print(a / b)        # 3.3333... — division always returns a float
print(a // b)       # 3        — floor division (integer result, rounded down)
print(a % b)        # 1        — modulo (remainder)
print(a ** b)       # 1000     — a to the power of b
```

Operator precedence is what you'd expect — parentheses force grouping:

```python
print(2 + 3 * 4)       # 14
print((2 + 3) * 4)     # 20
```

You can mix integers and floats freely; the result is a float:

```python
print(2 + 3.0)         # 5.0
```

### Exercise 2

Create `math.py`. Compute and print:

- The area of a circle of radius 5. (Area = π × r². Use `pi = 3.14159`.)
- How many whole hours and how many leftover minutes are in 200 minutes. (Hint: `//` and `%`.)

## 5. Strings

Strings hold text. Single quotes and double quotes are interchangeable — pick one and be consistent inside a project:

```python
name = "Alice"
greeting = 'Hello'
```

### Joining strings

Use `+` to glue strings together (this is called **concatenation**):

```python
first = "Field"
second = "Report"
title = first + " " + second
print(title)            # Field Report
```

You can only glue strings to strings. `"Alice" + 5` is an error — convert the number first with `str(5)`:

```python
n = 5
print("You have " + str(n) + " reports")   # works but verbose
```

### f-strings — the better way

The modern, readable way is an **f-string** — a string with `f` in front and `{...}` placeholders:

```python
name = "Alice"
n = 5
print(f"Hello {name}, you have {n} reports.")
# Hello Alice, you have 5 reports.
```

Anything between `{` and `}` is evaluated as a Python expression and dropped in:

```python
print(f"Doubled: {n * 2}")            # Doubled: 10
print(f"Total: {1 + 2 + 3}")          # Total: 6
```

### String basics

```python
name = "Alice"

print(len(name))            # 5 — number of characters
print(name.upper())         # ALICE
print(name.lower())         # alice
print(name + "!")           # Alice!
print("a" in name)          # False — case-sensitive
print("A" in name)          # True

quote = "Bread"
print(quote[0])             # B    — strings can be indexed like a list, starting at 0
print(quote[-1])            # d    — negative index counts from the end
```

You'll see f-strings constantly in HCPI code. Old code uses string concatenation or the older `%s` style — recognise them, but write f-strings yourself.

### Exercise 3

Create `strings.py`. Make variables for an outlet name, a price (a float), and a date (a string). Print an f-string like `"At Kikuubo Market on 2026-05-15, bread cost 1500.00 UGX."` using your three variables.

## 6. Booleans, comparisons, and truthiness

Booleans hold `True` or `False`. They come up whenever you compare things:

```python
print(5 > 3)            # True
print(5 == 5)           # True  — note: == compares, = assigns. Different!
print(5 != 6)           # True
print("a" == "A")       # False — string comparison is case-sensitive
```

Combine with `and`, `or`, `not`:

```python
age = 25
has_id = True

print(age >= 18 and has_id)             # True
print(age < 18 or not has_id)           # False
```

### Truthiness — the part to remember

In Python, **any value can be used as a true/false test**, not just booleans. The values that count as "false" (the **falsy** values) are:

- `False`
- `None`
- The number `0` (or `0.0`)
- The empty string `""`
- An empty list `[]`, dict `{}`, or set `set()`

Everything else is "true" (**truthy**). So:

```python
name = "Alice"
if name:
    print("There's a name")        # runs — non-empty string is truthy

items = []
if items:
    print("There are items")       # doesn't run — empty list is falsy
```

This shows up everywhere in HCPI code. `if record.outlet_id:` means "if this record has an outlet linked" — because an empty Many2one link evaluates as falsy.

### `None` and `is`

`None` deserves special treatment. To check "is this missing?" use `is None`, not `== None`:

```python
value = None

if value is None:
    print("missing")          # this is the idiomatic check

if value == None:
    print("missing")          # also works, but Python style guides discourage it
```

### Exercise 4

Create `booleans.py`. Make a variable `score = 75`. Print whether the score is a pass (≥ 50) using a comparison. Then make `name = ""` and print whether it's empty using only truthiness (no `==`).

## 7. Lists

A **list** is an ordered collection of values. Square brackets, comma-separated:

```python
items = ["bread", "milk", "eggs"]
prices = [1500, 3000, 4500]
mixed = ["bread", 1500, True]      # legal but unusual — usually lists hold one kind of thing
```

### Reading from a list

Items have positions (**indices**) starting at `0`:

```python
items = ["bread", "milk", "eggs"]

print(items[0])         # bread     — the first item
print(items[1])         # milk
print(items[-1])        # eggs      — negative index counts from the end
print(items[-2])        # milk
print(len(items))       # 3         — number of items
```

You can grab a **slice** — a sub-list:

```python
print(items[0:2])       # ['bread', 'milk'] — from index 0 up to (not including) 2
print(items[1:])        # ['milk', 'eggs']  — from 1 to the end
print(items[:2])        # ['bread', 'milk'] — from start to 2
```

### Modifying a list

```python
items = ["bread", "milk", "eggs"]

items.append("rice")          # add to the end       → ['bread', 'milk', 'eggs', 'rice']
items.insert(0, "salt")       # insert at index 0    → ['salt', 'bread', 'milk', 'eggs', 'rice']
items.remove("milk")          # remove first match   → ['salt', 'bread', 'eggs', 'rice']
del items[0]                  # remove by index      → ['bread', 'eggs', 'rice']
items[1] = "yoghurt"          # replace by index     → ['bread', 'yoghurt', 'rice']

print("bread" in items)       # True  — membership test
print("milk" in items)        # False
```

### Lists share their contents

This catches people. Assigning a list to another name doesn't copy it — both names point at the same list:

```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)        # [1, 2, 3, 4] — the same list! a and b are two names for one thing
```

If you want a real copy, use `a.copy()` or `list(a)`.

### Exercise 5

Create `lists.py`. Make a list of three outlet names. Add a fourth. Remove the second. Print the final list and its length.

## 8. Dictionaries

A **dictionary** (`dict`) maps **keys** to **values**. Curly braces, `key: value` pairs:

```python
outlet = {
    "name": "Kikuubo Market",
    "code": "K01",
    "active": True,
}
```

Think of it like a row in a spreadsheet where columns have names instead of positions.

### Reading and writing

```python
print(outlet["name"])         # Kikuubo Market
print(outlet["code"])         # K01

outlet["address"] = "Plot 4"  # add a new key
outlet["active"] = False      # update existing

del outlet["code"]            # remove a key
```

Accessing a key that doesn't exist crashes:

```python
print(outlet["email"])        # KeyError: 'email'
```

The safe alternative is `.get()`, which returns `None` (or a default you supply) when the key is missing:

```python
print(outlet.get("email"))            # None
print(outlet.get("email", "n/a"))     # n/a
```

### Looping over a dict

```python
outlet = {"name": "Kikuubo", "code": "K01", "active": True}

for key in outlet:
    print(key)                # name, code, active

for key, value in outlet.items():
    print(key, "=", value)    # name = Kikuubo, code = K01, active = True
```

`"name" in outlet` checks whether a **key** exists (not a value).

Dicts are everywhere in Odoo. Every time you create a record you pass a dict of field names to values:

```python
self.env['hcpi.outlet'].create({
    'name': 'Kikuubo',
    'code': 'K01',
})
```

### Exercise 6

Create `dicts.py`. Make a dict for a person with three keys: name, age, email. Print just the email. Use `.get()` to safely look up a key that doesn't exist. Add a fourth key. Loop over the dict and print each key/value pair.

## 9. Tuples and sets

Two other collection types, used less often than lists and dicts but worth recognising.

### Tuples — "fixed lists"

A **tuple** is like a list but immutable — once created, you can't change it:

```python
point = (10, 20)
print(point[0])           # 10
# point[0] = 99           # TypeError — can't change a tuple
```

You can **unpack** a tuple (or list) into multiple variables:

```python
x, y = point
print(x)      # 10
print(y)      # 20
```

In Odoo, **domains** are lists of tuples:

```python
domain = [('state', '=', 'active'), ('outlets_visited', '>', 0)]
```

Each `('state', '=', 'active')` is a 3-tuple — a fixed `(field, operator, value)` triple.

### Sets — uniqueness, fast lookup

A **set** holds unique values with no duplicates. Curly braces with no key-value structure:

```python
tags = {"food", "drink", "food"}    # duplicates auto-removed → {"food", "drink"}
tags.add("snack")
tags.discard("missing")              # no error if absent

print("food" in tags)                 # True
```

Use a set when you want "is this in the collection?" to be very fast (much faster than `in list` for thousands of items), or to deduplicate things.

## 10. `if`, `elif`, `else`

Programs need to make decisions. `if` lets you run a block of code only when a condition is true:

```python
score = 75

if score >= 90:
    print("A")
elif score >= 80:
    print("B")
elif score >= 50:
    print("Pass")
else:
    print("Fail")
```

Three rules:

1. **No parentheses around the condition.** `if score >= 90:` not `if (score >= 90):`.
2. **The colon at the end is required.**
3. **Indentation defines which lines belong to the `if`.** Convention is 4 spaces. The block ends when you stop indenting.

```python
if score >= 50:
    print("pass")        # part of the if block (indented)
    print("well done")   # part of the if block (still indented)
print("done")            # NOT part of the if block (back to no indent) — always runs
```

VS Code converts your Tab key to 4 spaces by default — let it. **Don't mix tabs and spaces** in the same file or Python will refuse to run it.

### `elif` and `else`

- `elif` (short for "else if") — checked only if the previous `if` was false.
- `else` — runs when none of the `if`/`elif` conditions were true. Optional.

### Exercise 7

Create `grade.py`. Set `score = 67`. Use `if/elif/else` to print one of: "A" (≥90), "B" (≥80), "C" (≥70), "Pass" (≥50), "Fail" otherwise.

## 11. Loops — `for` and `while`

### `for`

`for` walks through the items in a collection one at a time:

```python
items = ["bread", "milk", "eggs"]

for item in items:
    print(item)
```

The variable `item` takes the value of each list element in turn. The body is indented just like with `if`.

You'll often want a counted loop. `range(n)` produces `0, 1, 2, ..., n-1`:

```python
for i in range(5):
    print(i)            # 0, 1, 2, 3, 4

for i in range(2, 8):
    print(i)            # 2, 3, 4, 5, 6, 7

for i in range(0, 10, 2):
    print(i)            # 0, 2, 4, 6, 8 — third argument is the step
```

### `enumerate` — when you need both index and value

```python
items = ["bread", "milk", "eggs"]

for index, item in enumerate(items):
    print(index, item)
# 0 bread
# 1 milk
# 2 eggs
```

### Looping over a dict

```python
outlet = {"name": "Kikuubo", "code": "K01"}

for key, value in outlet.items():
    print(key, "=", value)
```

### `while`

`while` repeats as long as a condition stays true:

```python
n = 5
while n > 0:
    print(n)
    n = n - 1            # without this, the loop never ends
```

`break` exits a loop early; `continue` skips to the next iteration:

```python
for i in range(10):
    if i == 5:
        break             # stop the loop entirely when i hits 5
    if i % 2 == 0:
        continue          # skip the print for even numbers
    print(i)              # only odd numbers below 5: 1, 3
```

You'll write `for` loops far more often than `while`. Reach for `while` only when you genuinely don't know how many times to repeat.

### Exercise 8

Create `loops.py`. Make a list of five outlet names. Loop over them and print `"1. Kikuubo"`, `"2. Owino"`, etc. — number each line. (Hint: `enumerate`, and remember it starts at 0 so add 1.)

## 12. Functions

A **function** is a reusable block of code with a name. You **define** it once and **call** it whenever you want it to run.

```python
def greet(name):
    print(f"Hello, {name}")

greet("Alice")          # Hello, Alice
greet("Bob")            # Hello, Bob
```

Breaking that down:

- `def greet(name):` — defines a function called `greet` that takes one **parameter**, `name`.
- The indented block is the function's **body** — what runs when it's called.
- `greet("Alice")` — **calls** the function with the value `"Alice"`. Inside the function, `name` is now `"Alice"`.

### Returning a value

Functions can give a value back with `return`:

```python
def double(x):
    return x * 2

result = double(5)
print(result)           # 10
print(double(7))        # 14 — you can pass the return straight to print
```

A function without an explicit `return` returns `None`.

### Multiple parameters; default values

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}"

print(greet("Alice"))                       # Hello, Alice
print(greet("Alice", greeting="Hi"))        # Hi, Alice
print(greet("Alice", "Hey"))                # Hey, Alice — positional also fine
```

Parameters with defaults must come *after* parameters without defaults.

### Why functions matter

Functions let you:

- **Name a piece of behaviour** — so the reader knows what it does without reading every line.
- **Avoid repeating code** — write once, call from many places.
- **Test in isolation** — a small function is easier to reason about than a big script.

Every Odoo model method is a function. When you write `def action_submit(self):` you're defining a function that the framework calls when a user clicks a button.

### Exercise 9

Create `functions.py`:

- Write `def total(prices):` that takes a list of prices and returns the sum. (Hint: there's a built-in `sum()` you can use.)
- Write `def total_with_tax(prices, tax_rate=0.18):` that adds tax. The default rate is 18%.
- Call both with the list `[1500, 3000, 4500]` and print the results.

## 13. Modules and imports

A **module** is a `.py` file. You can use code from one module inside another by **importing** it.

Create `helpers.py`:

```python
def double(x):
    return x * 2

def shout(text):
    return text.upper() + "!"
```

In the same folder, create `main.py`:

```python
import helpers

print(helpers.double(5))            # 10
print(helpers.shout("hello"))       # HELLO!
```

Or pull specific names out:

```python
from helpers import double, shout

print(double(5))                    # 10
print(shout("hello"))               # HELLO!
```

Python also ships with built-in modules you can import. For example, `math`:

```python
import math

print(math.sqrt(16))                # 4.0
print(math.pi)                      # 3.141592...
```

In HCPI you'll see imports like:

```python
from odoo import fields, models, api
from odoo.exceptions import UserError, ValidationError
```

That's "from the `odoo` package, get `fields`, `models`, and `api`." Once you've read those, you understand half the line count of every HCPI model file.

### Exercise 10

Create `calc.py` with two functions: `add(a, b)` returning their sum, and `multiply(a, b)` returning their product. Create `use_calc.py` that imports both and prints `add(3, 4)` and `multiply(3, 4)`.

## 14. Classes — just enough for Odoo

A **class** is a blueprint for an **object** — a thing that bundles some data and some behaviour together. You'll write classes whenever you build a model in HCPI.

```python
class Outlet:
    def __init__(self, name, code):
        self.name = name
        self.code = code

    def label(self):
        return f"{self.code}: {self.name}"
```

Then use it:

```python
o = Outlet("Kikuubo", "K01")
print(o.name)               # Kikuubo
print(o.label())            # K01: Kikuubo
```

Reading the class line by line:

- `class Outlet:` — define a new class called `Outlet`.
- `def __init__(self, name, code):` — the **constructor**, called automatically when you write `Outlet(...)`. The double underscores make this name special.
- `self` — every method takes `self` as its first parameter. `self` is the object the method is being called on. You don't pass it when calling (`o.label()`, not `o.label(o)`) — Python fills it in.
- `self.name = name` — store the parameter on the object so other methods can read it.
- `def label(self):` — a regular method. Can access `self.name` and `self.code`.

### Why `self`?

If you have two outlets:

```python
a = Outlet("Kikuubo", "K01")
b = Outlet("Owino", "OW1")

print(a.label())            # K01: Kikuubo
print(b.label())            # OW1: Owino
```

Both call the same `label` method, but each gets a different result. `self` is how `label` knows which object it's working with — when you call `a.label()`, Python passes `a` in as `self`; when you call `b.label()`, it passes `b`.

### What HCPI models look like

```python
from odoo import fields, models

class Outlet(models.Model):
    _name = 'hcpi.outlet'
    _description = "Outlet"

    name = fields.Char(required=True)

    def label(self):
        return f"{self.name}"
```

Now you can read this. It says: define a class `Outlet` that inherits from `models.Model` (a class Odoo provides). Give it two settings (`_name`, `_description`). Declare a field called `name`. Give it a method called `label`.

Two extra things specific to Odoo:

1. **`name = fields.Char(...)`** assigns a Field *object* to a class attribute. Odoo reads these on install and creates database columns from them.
2. **`self` in an Odoo method is a recordset** — a list-like collection of records, not a single object. So `self.name` only makes sense if `self` contains exactly one record. You'll learn the recordset patterns in [Part 1 of the module tutorial](../module/part1-models.md).

That's all the class theory you need to start.

### Exercise 11

Create `classes.py`. Define a class `Report` with `__init__` that takes a `name` and a `date`, and a method `summary()` that returns `f"Report {self.name} on {self.date}"`. Create two instances and print each one's summary.

## 15. Errors and exceptions

When something goes wrong — a missing key, a type mismatch, a division by zero — Python **raises an exception**. By default, an exception stops the program.

```python
x = 10 / 0
# ZeroDivisionError: division by zero
print("never reached")
```

You can **handle** an exception with `try` / `except`:

```python
try:
    x = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero")
    x = 0

print(x)            # 0
```

You can catch specific types:

```python
try:
    n = int("not a number")
except ValueError as e:
    print(f"Bad number: {e}")
```

### Raising your own

You can deliberately raise an exception when something is wrong:

```python
def set_price(p):
    if p < 0:
        raise ValueError("Price cannot be negative")
    return p

set_price(10)           # fine
set_price(-3)           # raises ValueError
```

In HCPI you'll see two Odoo-specific exception types:

```python
from odoo.exceptions import UserError, ValidationError

if price < 0:
    raise UserError("Price must be non-negative")
```

`UserError` and `ValidationError` show their message to the user as a dialog. You usually *raise* these, not *catch* them — let them bubble up to the UI.

## 16. List comprehensions — the most "Pythonic" thing

A **list comprehension** is a one-line way to build a list from another collection. It's the single biggest syntax difference from most other languages and it's all over Odoo code, so spend a couple of minutes here.

The long way:

```python
numbers = [1, 2, 3, 4, 5]

doubled = []
for n in numbers:
    doubled.append(n * 2)

print(doubled)          # [2, 4, 6, 8, 10]
```

The short way — a list comprehension:

```python
doubled = [n * 2 for n in numbers]
print(doubled)          # [2, 4, 6, 8, 10]
```

Read it left-to-right: **"\[take `n * 2`, for each `n` in `numbers`\]"**.

You can also **filter**:

```python
evens = [n for n in numbers if n % 2 == 0]
print(evens)            # [2, 4]
```

"\[take `n`, for each `n` in `numbers`, if `n` is even\]"

### Dict comprehensions

Same idea but for dicts:

```python
numbers = [1, 2, 3, 4]
squared = {n: n * n for n in numbers}
print(squared)          # {1: 1, 2: 4, 3: 9, 4: 16}
```

### Where you'll see them in HCPI

All the time:

```python
# Pull all the names out of a recordset
names = [outlet.name for outlet in outlets]

# Map outlets by id
by_id = {outlet.id: outlet for outlet in outlets}

# Keep only active ones
active = [o for o in outlets if o.active]
```

The first time these look strange. After 20 of them they feel normal.

### Exercise 12

Create `comprehensions.py`:

- Given `nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]`, build a list of the squares of the even numbers using one comprehension.
- Given `names = ["alice", "BOB", "Charlie"]`, build a list with each name lower-cased.

## 17. Lambdas — tiny anonymous functions

A `lambda` is a one-line function with no name. Use it when you need a small function once, on the spot:

```python
square = lambda x: x * x
print(square(5))            # 25

# More commonly, used inline as an argument:
items = [(1, "apple"), (3, "banana"), (2, "cherry")]
items_sorted = sorted(items, key=lambda pair: pair[0])
print(items_sorted)         # [(1, 'apple'), (2, 'cherry'), (3, 'banana')]
```

You'll see lambdas in Odoo for two things:

**Field defaults that need to be computed at create time:**

```python
date = fields.Date(default=lambda self: fields.Date.today())
```

That `lambda self:` is a function Odoo calls each time a new record is created. It runs `fields.Date.today()` *then*, not when the file is first loaded.

**Filtering recordsets:**

```python
high_severity = records.filtered(lambda r: r.severity == 'high')
```

## 18. Decorators — recognising `@api.depends`

A **decorator** is a `@something` line just above a function definition. It modifies how the function behaves. You don't need to write your own decorators yet — but you need to recognise them, because Odoo uses them on every computed field.

```python
@api.depends('name', 'code')
def _compute_full_label(self):
    for record in self:
        record.full_label = f"{record.code}: {record.name}"
```

The decorator `@api.depends('name', 'code')` is Odoo's way of saying "this method computes a field; re-run it whenever `name` or `code` changes."

Other Odoo decorators:

| Decorator | Means |
|---|---|
| `@api.depends(...)` | Re-run this compute when listed fields change. |
| `@api.constrains(...)` | Validate on save; raise if invalid. |
| `@api.onchange(...)` | Update the UI as the user types — doesn't save anything. |
| `@api.model` | This method operates on the model class itself, not on records. |

You'll see all four in the module tutorial.

## 19. A few idioms HCPI code uses

A grab-bag of patterns that look strange the first time:

**Ternary expression (one-line if/else for an assignment):**

```python
label = "active" if record.active else "archived"
```

Same as:

```python
if record.active:
    label = "active"
else:
    label = "archived"
```

**Safe dict lookup with default:**

```python
# Instead of:
if "name" in record:
    name = record["name"]
else:
    name = "unknown"

# Pythonic:
name = record.get("name", "unknown")
```

**`with` for resources that need cleanup:**

```python
with open("file.txt") as f:
    content = f.read()
# file is automatically closed when the with-block exits
```

**`zip` to pair up two lists:**

```python
names = ["a", "b", "c"]
prices = [10, 20, 30]
for name, price in zip(names, prices):
    print(name, price)
```

## What you should be able to do now

After working through this page you should be able to:

- Create, save, and run a Python file from VS Code's integrated terminal in WSL.
- Use variables, basic types, lists, dicts, tuples, sets.
- Write `if`/`elif`/`else` and `for`/`while` loops.
- Define functions with parameters and return values.
- Import code from one file into another.
- Read a small class with `__init__` and methods.
- Read a list or dict comprehension without translating it line-by-line in your head.
- Spot decorators, lambdas, and f-strings as you go.

If any of these still feels shaky, the cure is **practice**, not more reading. The exercises take ~30 minutes if you do them in order, and they're cumulative — each builds slightly on the last.

## What's next

The next step applies all of this to a real Odoo module — you'll write models, views, security rules, and reports.

➡️ **[Building HCPI Field Reports — Part 1](../module/part1-models.md)** — start with models, fields, and the ORM.

## Cheat sheet

| You want to… | You write… |
|---|---|
| Show something | `print(x)` |
| Make a comment | `# like this` |
| Assign a variable | `x = 5` |
| Compare for equality | `==` (not `=`) |
| Check missing value | `if value is None:` |
| Build a string with values inside | `f"Hello {name}"` |
| Get length | `len(x)` |
| Sum a list | `sum(my_list)` |
| Walk a list | `for item in items:` |
| Counted loop | `for i in range(n):` |
| Walk a dict | `for k, v in my_dict.items():` |
| Define a function | `def f(x):` then indented body |
| Default argument | `def f(x, n=1):` |
| Make a class | `class Name:` |
| Constructor | `def __init__(self, ...):` |
| Inside a method | always use `self.field` to access attributes |
| Build a list from another | `[x * 2 for x in items]` |
| Filter a list | `[x for x in items if x > 0]` |
| Anonymous function | `lambda x: x * 2` |
| Import a file | `from helpers import func` |
| Catch an error | `try: ... except SomeError: ...` |
| Raise an error | `raise ValueError("bad")` |
