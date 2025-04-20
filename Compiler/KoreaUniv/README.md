# COSE-312
https://prl.korea.ac.kr/courses/cose312/2025/

## Index

## Lecture 1
Compiler: Software that **tranlates** source language into target language <br>

Target language is usually machine language(x86), but not always <br>

Java => First compiles into intermediate form named **bytecodes** 

![Interpreter](./images/Lec1_1.png)

Compiler should preserve the meaning(semantics) of the source program

![Semantics](./images/Lec1_2.png)

### Structure of Compilers

![Structures](./images/Lec1_3.png)

### Front End
Input : source program(character string) => Translate into **Intermediate Representation(IR)** <br>

IR: 2-dimensional structure. Abstract-Syntax-Tree <br>

#### Lexical Analyzer
Character Stream -> Token Stream <br>

Token: Defined as pair of **Type** and **Value**

![Token](./images/Lec1_4.png)

#### Syntax Analyzer
Token Stream -> Syntax Tree <br>

With Syntax Tree, semantics of the program appears

![Syntax_Tree](./images/Lec1_5.png)

#### Semantic Analyzer
Check semantic correctness of input program(very hard) <br>

Syntactic correctness can be checked perfectly with parser <br>

Semantic correctness is much more complicated and tricky <br>

Type checking is important. Make sure every operator has **matching operand**

![Semantic](./images/Lec1_6.png)

#### IR Translator
Syntax Tree -> IR (IR is also tree structure) <br>

Intermediate Representation:
- lower-level than source language
- higher-level than target language

Three-Address Code -> At most two operand on right side <br>

Why using IR? -> Reuse, Optimization...

#### Optimizer
Consists of multiple optimizers

![Optimizer](./images/Lec1_7.png)

Let's look at a simple example

![Optimizer_Exmple](./images/Lec1_8.png)

Following features have been applied

- Constant propagation: Substitute subject of allocation into value itself
- Deadcode elimination: Delete unnecessary code 
- Copy elimination

### Back End
IR -> Target machine code <br>

Difference between high-level and low-level: <br>

Register Allocation Problem: cannot use many variables in machine code <br>

But this problem is NP, solvale at least

### Summary
![summary](./images/Lec1_9.png)

