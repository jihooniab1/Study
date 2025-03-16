# Ocaml

## Index
- [1 Ocaml Basic](#ocaml-basic)
    -[1.1 Basic Operation](#basic-operation)
    -[1.2 Conditional Expression](#conditional-expression)
    -[1.3 Variable](#variable)
    -[1.4 Function](#function)
    -[1.5 Static Type System](#static-type-system)
    -[1.6 Pattern Matching](#pattern-matching)
    -[1.7 Tuple & List](#tuple--list)
    -[1.8 User defined type](#user-defined-type)
    -[1.9 Exception Handle](#exception-handle)
    -[1.10 Module](#module)
- [2 Recursive Function](#recursive-function)
    -[2.1 Append list](#append-list)
    -[2.2 Flipping a list](#flipping-a-list)
    -[2.3 Finding nth element](#finding-nth-element-of-the-list)
    -[2.4 Deleting certain element](#deleting-certain-element-that-first-appears)
    -[2.5 Insertion Sort](#insertion-sort)
- [3 Higher order function](#higher-order-function)
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

### Conditional expression
Ocaml supports conditional expression
```
if e1 then e2 else e3

if 2 > 1 then 0 else 1;;
```
e1 => Must be boolean expression <br>

e2, e3 => Must have same types of value <br>

### Variable
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

### Function
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

### Static Type System
OCaml => Statically typed language + Sound type system <br>

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
Type error occurs 

### Pattern Matching
With pattern matching, we can express nested condition concisely 
```
let is123 s = if s = "1" then true
              else if s = "2" then true
              else if s = "3" then true
              else false

let is123 s =
    match s with
    "1" -> true
   |"2" -> true
   |"3" -> true
   | _  -> false

let is123 s =
    match s with
    "1" | "2" | "3" -> true
    | _ -> false
```
When using nested pattern matching, you'd better to enclose the nested **match...with** in parentheses. <br>

```
match x with
| 1 -> true
| 2->
    (match y with
        | "a" -> true
        | "b" -> false)
| 3 -> false

match x with
| 1 -> true
| 2 ->
  begin
    match y with
    | "a" -> true
    | "b" -> false
  end
| 3 -> false
```

### Tuple & List
Tuple: Collection of values. ex: (1, "one), (2, "two", true) <br>

```
let x = (1, "one");;
val x : int * string = (1, "one")
```

To access each element of tuple, you can use pattern matching <br>
Below is functions that fetch the first/second element from tuple. 
```
let fsp p = match p with (x,_) -> x;;
val fst : 'a * 'b -> 'a = <fun>

let snd p = match p with (_, x) -> x;;
val snd : 'a * 'b -> 'b = <fun>

let fst (x,_) -> x;;
val fst : 'a * 'b -> 'a = <fun>
```

**'a * 'b** => Function fst is a polymorphic function that returning arbitrary 'a type value after getting 'a * 'b type of tuple <br>

Also you can use tuple pattern in **let** not only in argument
```
let p = (1, true);;
val p : int * bool = (1, true)

let (x,y) = p;;
val x : int = 1
val y : bool = true
```

List: Sequence of elements of the same type <br>

List use **;** to distinguish elements. [1, 2, 3] => List that has a tuple ( 1, 2, 3) as element
```
[1, 2, 3];;
- : (int * int * int) list = [(1, 2, 3)]

[1; 2; 3];;
- : int list = [1; 2; 3]
```
Empty list => **[]**, polymorphic **'a list** <br>

In OCaml, every element in a tuple should have the same type, And order between elements matters
```
[1;2;3] = [2;3;1];;
- : bool = false
```

**Head**: First element of list, Rest is **tail** <br>

Suppose list of type **t list** => head type is **t** and tail is **t list** <br>

Every type can be an element of a list. Below is also possible
```
[[1;2;3]; [4]; []];;
- : int list list = [[1; 2; 3]; [4]; []]
```

There are two basic list operation that OCaml supports

1. **::(cons)** => Add a element in front of a list
```
1::[2;3];;
- : int list = [1; 2; 3]

1::2::3::[];;
- : int list = [1; 2; 3]
```

2. **@(append)** => Append two lists 
```
[1; 2] @ [3; 4; 5];;
- : int list = [1; 2; 3; 4; 5]
```

When handling list, pattern match is used a lot. Below is functions that returns head and tail of list 
```
let hd l = 
    match l with
    | [] -> raise (Failure "hd is undefined")
    | a::b -> a;;
val hd : 'a list -> 'a = <fun>

let tl l = 
    match l with
    | [] -> raise (Failure "tl is undefined")
    | a::b -> b;;
val tl : 'a list -> 'a list = <fun>
```
Also this is possilbe, but warning message occurs
```
let hd (a::b) = a;; -> Non exhaustive warning
```

We can make a function that returns the length of a list
```
let rec length l =
    match l with
    | [] -> 0
    | h::t -> 1 + length t;;
val length : 'a list -> int = <fun>
```

### User-defined type
New type can be defined with **type** keyword. Also can make a new name to existing type
```
type var = string
type vector = float list
type matrix = float list list
```

Or, we can make a collection of new values and define it as a type
```
type days = Mon | Tue | Wed | Thu | Fri | Sat | Sun;;

Mon;;
- : days = Mon

Tue;;
- : days = Tue
```
Mon..Sun => **Constructor**, meaning each different value of days type <br>

In OCaml, name of type starts with lowercase, constructor with uppercase
```
let nextday d =
match d with
| Mon -> Tue | Tue -> Wed | Wed -> Thu | Thu -> Fri 
| Fri -> Sat | Sat -> Sun | Sun -> Mon;;
```

We can also define constructors to have other value
```
type shape = Rect of int * int | Circle of int;;

Rect (2, 3);;
- : shape = Rect (2, 3)

Circle 5;;
- : shape = Circle 5
```

With this, we can make a function that calculates the width of defined shape
```
let area s = 
match s with
| Rect (w,h) -> w * h
| Circle r -> r * r * 3;;
val area : shape -> int = <fun>
```

Let's define a type named **intlist**
```
type intlist = Nil | Cons of int * intlist;;
type intlist = Nil | Cons of int * intlist

Nil;;
- : intlist = Nil

Cons (1, Nil);;
- : intlist = Cons (1, Nil)

Cons (1, Cons (2, Nil));;
- : intlist = Cons (1, Cons (2, Nil))
```

Cons (1, Cons (2, Nil)) => [1;2] <br>

We can make a function that returns the length of list
```
let rec length l =
    match l with
    | Nil -> 0
    | Cons (_, l') -> 1 + length l';;
val length : intlist -> int = <fun>
```

Let's define another one
```
type exp =
      Int of int
    | Minus of exp * exp
    | Plus of exp * exp
    | Mult of exp * exp
    | Div of exp * exp
```

For example, **(1+2)*(3/3)** is
```
Mult(Plus(Int 1, Int 2), Div(Int 3, Int 3));;
```

And Inductive Rule that denotes the meaning of exp can implemented like this
```
let rec eval exp =
    match exp with
    | Int n -> n
    | Plus (e1, e2) -> (eval e1) + (eval e2)
    | Mult (e1, e2) -> (eval e1) * (eval e2)
    | Minus (e1, e2) -> (eval e1) - (eval e2)
    | Div (e1, e2) ->
      let n1 = eval e1 in
      let n2 = eval e2 in
        if n2 <> 0 then n1 / n2
        else raise (Failure "division by 0");;
val eval : exp -> int = <fun>
```

### Exception Handle
When run-time error occurs -> **exception** happen <br>
```
let div a b = a / b;;
val div : int -> int -> int = <fun>

div 10 0;;
Exception: Division_by_zero
```

Can use **try ... with** to handle runtime exception
```
let div a b =
    try
        a / b
    with Division_by_zero -> 0;;
val div : int -> int -> int = <fun>
```

Can register new exception using **exception** keyword
```
exception Fail;;
exception Fail

let div a b =
    if b = 0 then raise Fail
    else a / b;;
val div : int -> int -> int = <fun>
```

### Module
Module: Collection of types and values <br>

Let's define **queue** data structure with module
```
module IntQueue = struct
    type t = int list
    exception E
    let empty = []
    let enq q x = q @ [x]
    let is_empty q = q = []
    let deq q =
        match q with
        | [] -> raise E
        | h::t -> (h, t)
    let rec print q =
        match q with
        | [] -> print_string "\n"
        | h::t -> print_int h; print string " "; print t
end
```

User can use queue without knowing the detail of implementation
```
let q0 = IntQueue.empty
let q1 = IntQueue.enq q0 1
let q2 = IntQueue.enq q1 2
let (_,q3) = IntQueue.deq q2
let _ = IntQueue.print q1
```

## Recursive Function
Let's practice

### Append list
```
append [1; 2; 3] [4; 5; 6; 7];;
- : int list = [1; 2; 3; 4; 5; 6; 7]
```

**append** gets two input: l1, l2 and returns l1@l2 <br>

To recursively implement this, <br>

- If l1 is empty => Just append l1, l2
- If l1 is nonempty => Devide l1 into head and tail, recursively calculate appended list with tail, l2 and then append l1
```
let rec append l1 l2 =
    match l1 with
    | [] -> l2
    | hd :: tl -> hd :: (append tl l2)
```

### Flipping a list
```
reverse [1; 2; 3];;
- : int list = [3; 2; 1]
```

Just flip tail and then connect head at the back
```
let rec reverse l =
    match l with
    | [] -> []
    | hd::tl -> (reverse tl)@[hd]
```

### Finding Nth element of the list
```
nth [1;2;3] 0;;
- : int = 1

nth [1;2;3] 3;;
Exception: Failure "list is too short"

let rec nth l n =
    match l with
    | [] -> raise (Failure "list is too short")
    | hd :: tl ->
        if n = 0 then hd
        else nth tl (n - 1);;
```
### Deleting certain element that first appears
```
remove_first 2 [1; 2; 3];;
- : int list = [1; 3]
remove_first 4 [1; 2; 3];;
- : int list = [1; 2; 3]

let rec remove_first n l =
    match l with
    | [] -> []
    | hd::tl ->
        if hd = n then tl
        else hd::remove_first n tl;;
```

### Insertion Sort
```
insert 2 [1;3];;
- : int list = [1; 2; 3]
insert 4 [];;
- : int list = [4]

let rec insert n l =
    match l with
    | [] -> [n]
    | hd::tl ->
        if n <= hd then n::hd::tl
        else hd::(insert a tl)
```
With this **insert** function, we can also define sort function

```
let rec sort l =
    match l with
    | [] -> []
    hd::tl -> insert hd (sort tl)
```

## Higher-order function
Higher-order => Input, output with other function. Enables abstraction and reuse <br>

### map
