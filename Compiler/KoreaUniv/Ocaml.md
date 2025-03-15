# Ocaml

## Index
- [1.1 Ocaml Basic](#ocaml-basic)

## Ocaml Basic
```
let _ = print_endline "Hello World"
```
This is "Hello World" in Ocaml <br>

```
#use "hello.ml"
```
Can use **#use** to execute entire file <br>

```
ocamlc hello.ml

./a.out
```
Can compile, execute Ocaml file <br>

### Basic Operation
```
1 + 2 * 3;;
```

Integer calculation <br>
```
1.1 + 2.2 * 3.3;;
```
Float calculation <br>

```
3 + 2.0;;
```
This occurs error => **type case** <br>

```
3 + (int_of_float 2.0);;
(float_of_int 3) + 2.0;;
```

Boolean expression is used like this
```
true;;
=> bool = true
false;;
=> bool = false

1 = 2;;
=> bool = false

1 <> 2;;
=> bool = true

2 <=(1+1);;
=> bool = true

not (2 > 1);;
=> bool = false;;
```

Also Ocaml supports character, string, unit 
```
'c';;
=> char = 'c'

"Objective " & "Caml";; (^ is concatenation)
=> string = "Objective Caml"

();;
=> unit = ()
```
**unit** => Type that has only one value, denoting as **()** (similar with void) <br>

Ocaml supports conditional expression
```
if e1 then e2 else e3

if 2 > 1 then 0 else 1;;
```
e1 => Must be boolean expression <br>

e2, e3 => Must have same types of value <br>

Ocaml supports variable
```
let x = 3 + 4;;
```
This makes a kind of global variable. If we want make a local variable, we can use **let...in**
```
let x = e1 in e2
```
This means, let **x** have value e1, and calculate e2. <br>

At this moment, variable **x**'s scope is e2

```
let a = 1 in a * 2;;
```
Variable **a** only exists in **a*2**. Also this expression can be used multiple times <br>

```
let d =
    let a = 1 in
    let b = a + a in
    let c = b + b in
        c + c;;

val d : int = 8

# d;;
- : int = 8
# a;;
Error: Unbound value a
# b;;
Error: Unbound value b
# c;;
Error: Unbound value c
```
We defined global variable **d** with nested **let**. a, b, c is local variable <br>

Ocaml alseo use **let** when defining function
```
let square x = x * x;;

val square : int -> int = <fun>
```

(function name) (argument) **=** (function body) <br>

Can use like this
```
sqaure 2;;
- : int = 4

square (-3);;
- : int = 9
```

Function square can also be defined like below
```
let square = fun x -> x * x;;
val square : int -> int = <fun>
```

```
fun x -> x * x
```
This means function that returns **x * x** after getting argument **x**<br>

After defining **Anonymous function**, we gave name square <br>

Function body can be a conditional expression 
```
let neg x = if x < then true else false;;
```

In Ocaml, functions are usually used in the form of **e1 e2** <br>

and e1, e2 can be a arbitrary expression. Below expression is also possible
```
(fun x -> x * x) 3;;
- : int = 9
```

Functions can have multiple arguments
```
let sum_of_squares x y = (square x) + (square y);;
val sum_of_squares : int -> int -> = <fun>
```

