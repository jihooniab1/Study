# COSE-312
https://prl.korea.ac.kr/courses/cose312/2025/

## Index
- [1. Lecture 1](#lecture-1)
- [2. Lecture 2](#lecture-2)
- [3. Lecture 3](#lecture-3)
- [4. Lecture 4](#lecture-4)

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

## Lecture 2
Character sequence into token sequence based on **white space** <br>

parenthesis, char(keyword), star, if -> Thing like this are counted separately <br>

```
float match0 (char *s) /* find a zero */
{if (!strncmp(s, "0.0", 3))
    return 0.0;
}
```

Given this code...

![lex](./images/Lec2_1.png)

EOF: End Of File <br>

To do lexing...
- Specification: How to specify lexical patterns (Regular Expression)
- Recognition: How to **recognize** lexical patterns (DFA)
- Automation: How to automatically generate string recognizer from spec
(Thompson's construction and subset construction)

### Specification
#### Alphabet
Alpabet Σ is a **finite, non-empty** set of symbols <br>

#### Strings
String: finite sequence of symbols chosen from alphabet <br>

For example: 1, 01, 10110 are strings over Σ = {0, 1}

![string](./images/Lec2_2.png)

#### Languages
Language L is a subset of Σ∗: L ⊆ Σ∗

![language](./images/Lec2_3.png)

#### Regular Expressions
Meta language that specifies another language <br>

Since regular expression is language, it has syntax and semantics <br>

Syntax: Usually define with **context free grammar**
![syntax](./images/Lec2_4.png)
- Regular expressions (R) are defined recursively, with simpler regular expressions as components.
- The set of strings described by a regular expression forms a regular language, which is a specific type of formal language that can be recognized by a finite automaton. <br>

Semantics: L(R) => Meaning of Regular Expression R. L(R) is subset of Σ*
![semantic](./images/Lec2_5.png)

#### Regular Definitions
It's inconvenient to express all the specifications with Regular Expressions <br>

Regular Definition: Give names to regular expressions and use the names in subsequent expressions <br>

![reg_def](./images/Lec_6.png)

#### Extensions of Regular Expressions
![extension](./images/Lec2_7.png)

### String Recognition by Finite Automata
Implement recognized pattern with finite automata

#### NFA
NFA eventually means set of strings that NFA accepts(recognizable strings) <br>

Lexical Analyzer: Change string into equivalent NFA -> DFA <br>

![nfa](./images/Lec2_8.png)
-2Q -> Power set of Q. Set of every subset of Q
-if Q={q0, q1, q2}, 2^Q =  { {}, {q0}, {q1}, {q2}, {q0,q1}, {q0,q2}, {q1,q2}, {q0,q1,q2}}

#### DFA
DFA is special case of NFA
- No moves on ϵ
- For each state and input symbol, next state is unique 

![dfa](./images/Lec2_9.png)

### Automation
RE --- Thompson's contruction ---> NFA --- subset construction ---> DFA <br>

#### Principles of Compilation
Every automatic compilation 
- is done **compositionally** and
- maintains some **invariants** during compilation 

If program looks like this: R1|R2 <br>
- Compilation of R1|R2 is defined in terms of compilation of R1 and R2
- Compiled NFAs for R1 and R2 satisfy the invariants:
    - NFA has only one accepting state
    - no arc into initial state, and out of accepting state 

Source Language:
![source](./images/Lec2_10.png)

#### Compilation(Base Cases)
![base1](./images/Lec2_11.png)
![base2](./images/Lec2_12.png)
![base3](./images/Lec2_13.png)
![base4](./images/Lec2_14.png)

#### ϵ-Closures
ϵ-Closure(I) => Set of states reachable from I without consuming any symbols <br>

#### Running Example
1. Initial DFA state is d0 = ϵ0Closure({0})
![first](./images/Lec2_15.png)
2. For initial state S, consider every x ∈ Σ and compute corresponding next state
![second](./images/Lec2_16.png)
3. For the state {1,2,3,4,6,9}, compute next states
![third](./images/Lec2_17.png)

Continue until nothing changes <br>

Those states that include NFA's accepting state becomes DFA's accepting state.

#### Subset Construction Algorithm
![algorithm](./images/Lec2_18.png)

#### Algorithm for computing ϵ-Closures
Formal definition: T = ϵ-Closure(I) is the smallest set such that
![algo1](./images/Lec2_19.png)
Alternatively, T is the smallest solution of the equation F(X) ⊆ (X) where
![algo2](./images/Lec2_20.png)

Such solution is called the **least fixed point of F** <br>

Following is fixed point iteration algorithm 

![fixed](./images/Lec2_21.png)

## Lecture 3
Build parse tree (syntax tree) 

### Context-Free Grammar
Regular is weak to express 2-dimensional structure. Have ot use Context-Free Grammar<br>

Palindrome(w∈{0,1}∗|w=wR) is representative example of CFG. Not regular but context-free <br>

Every context-free grammar is defined by recursive definition 
- Bases: ϵ, 0, 1 -> palindromes
- Induction: If w is palindrome, so are 0w0, 1w1
![cfg](./images/Lec3_1.png)

### Derivation 
![derivation](./images/Lec3_2.png)

To check if a string is contained in some grammar, can do it by derivation <br>

From the start variable, if the string can be derived, that string is contained in grammar <br>

Parsing => Doing derivation in reverse 

![def](./images/Lec3_3.png)

- Leftmost Derivation(=>l): **Leftmost non-terminal** in each sentential is always chosen
- Rightmost Derivation(=>r): **Rightmost non-terminal** in each sentential is always chosen 

If S ⇒∗l α,α is left sentential form. =>*r -> right sentential form

### Parse Tree
Graphical tree-like representation of derivation <br>

Parse tree ignores variations in the order in which symbols are replaced <br>

```
E ⇒ −E ⇒ −(E) ⇒ −(E + E) ⇒ −(id + E) ⇒ −(id + id)
E ⇒ −E ⇒ −(E) ⇒ −(E + E) ⇒ −(E + id) ⇒ −(id + id)
```

![parse](./images/Lec3_4.png)

If derivation use same set of rules -> parse tree is identical 

### Ambiguity
Grammar is **ambiguous** if
- Produces more than one parse tree for some sentence
- Has multiple leftmost derivations
- Has multiple rightmost derivations

### Eliminating Ambiguity
E → E + E | E ∗ E | (E) | id -> Make layers of grammar 

1. Precedence: bind * tighter than +
1 + 2 * 3 => Always parse by 1 + (2 * 3)
2. Associativity: * and + associate to the left
1 + 2 + 3 => Always parse by (1 + 2) + 3

### Eliminating Left-Recursion
E -> E + T | T : left-recursive form <br>

Rewrite grammar using right recursion

```
E -> T E'
E' -> + T E'
E' -> ϵ
```

In general,
```
A → Aα | β 
```

can change like below
```
A → βA′
A′ → αA′| ϵ
```

### Left Factoring
```
S → if E then S else S
S → if E then S
```
This grammar has rules with the same prefix. => **Left Fator**

```
S → if E then S X
X → ϵ
X → else S
```

In general, if A → αβ1 | αβ2 are two A-productions, can refactor like this
```
A → αA′
A′ → β1 | β2
```

### Non-Context-Free Language Constructs
Case 1:
![ex1](./images/Lec3_5.png)

First w => Declaration of identifier w <br>
Second w => Use of identifier <br>

This problem is about checking identifier before using it <br>

However, CFG cannot handle this <br>

Case 2:
![ex2](./images/Lec3_6.png)

Modeling of checking function's parameter number <br>

Checking these properties is usually done during **semantic-analysis** 

## Lecture 4
Top-Down Parsing : Constructing parse tree starting from the root -> leaf <br>

Key Problem: When doing leftmost derivation, which rule should be selected? 

Expression Grammar:
```
E → T E′
E′ → + T E′| ϵ
T → F T′
T′ → ∗ F T′| ϵ
F → (E) | id
```

### Parsing Table
Parsing table for expression grammar looks like this
![table](./images/Lec4_1.png)
- $ -> endmarker

Sequence of predictive parsing for **id + id * id**
![ex](./images/Lec4_2.png)

Following is predictive parsing algorithm
- Input: string w and paring table M for grammar G
- Output: leftmost derivation of w or error
![parsingalgo](./images/Lec4_3.png)

