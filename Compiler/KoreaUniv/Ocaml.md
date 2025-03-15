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
OCaml denotes the type of function as => **int -> int -> int** <br>

This type can also be interpreted as **int -> (int -> int)** <br>
```
let f = sum_of_squares 3;;
val f : int -> int = <fun>

f 4;;
- : int = 25
```
When we define variable f as the value of **sum_of_squares 3**, it becomes the function that returns (squares 3) + (square y) with input y <br>

Recursive function have to use **rec** keyword to define
```
let rec factorial a =
    if a = 1 then 1 else a * factorial (a - 1);;
val factorial : int -> int = <fun>
```

Ocaml is a language that handle function as **first-clase** <br>

First-class value => <br>
- Can store the value in the variable 
- Can pass as the argument
- Can return as the return value of function 

```
let sum f a b = 
    (if f a then a else 0) + (if f b then b else 0)
val sum : (int -> bool) -> int -> int -> int = <fun>

let even x = x mod 2 = 0;;
val even : int -> bool = <fun>

sum even 3 4;;
- : int 4
```

A function can also return another function
```
let plus a = fun b -> a + b;;
val plus : int -> int -> int = <fun>

let plus 1 = plus 1;;
val plus1 : int -> int = <fun>
```
Like **sum** or **plus**, functions that input/output other functions are called **higher-order function** <br>

OCaml => Staticalaly typed language + Sound type system <br>

OCaml also guesses type during static type check <br>
```
lef f n =
    let a = 2 in
        a * n;;
val f : int -> int = <fun>
```

Let's recall function sum
```
let sum f a b =
    (if f a then a else 0) + (if f b then b e 0)
val sum : (int -> bool) -> int -> int -> int = <fun>
```
OCaml guesses type with following procedure <br>

- In conditional expression **if e1 then e2 else e3**, type of e2 and e2 should be same => a, b is integer <br>

**f** is used in the form of function call, and argument is given as **a** or **b**, and result of **f a** is used as e1 of contional expression <br>

So, we can guess the type of function f => int -> bool <br>

Lastly, sum returns the result of addition => returns int <br>

This procedure is called **automatic type inference** <br>

User can write type on his hand. In this case, OCaml checks if it is correct
```
let sum (f : int -> bool) (a: int) (b: int) : int
    = (if f a then a else 0) + (if f b then b else 0);;
```

In some cases, type of an expression can not de determined
```
let id x = x;;
val id : 'a -> 'a = <fun>

id 1;;
- : int = 1

id "abc";;
- : string = "abc"
```
When function operates with arbitrary type => Use type variable like **'a** to denote type <br>

We call these types **polymorphic type** => **polymorphic function** <br>

OCaml only supports polymorphic function that is defined with **let** => **let-polymorphic type system** <br>
```
let f = fun x -> x in
    let x = f 1 in
        let y = f true in
            3;;
- : int = 3
``` 

But, if we define the same function without let..
```
(fun f ->
    let x = f 1 in
        let y = f true in
            3) (fun x -> x);;
Error: The expression has type bool but an expression
was expected of type int
```
