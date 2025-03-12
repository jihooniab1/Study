# Summary for The Art, Science, and Engineering of Fuzzing: A Survey

## Index
- [1. Introduction](#introduction)
- [2. Systemization, Taxonomy, And Test Programs](#systemization-taxonomy-and-test-programs)
    [2.1 Fuzzing & Fuzz Testing](#fuzzing--fuzz-testing)
- [3. Preprocess]
- [4. Scheduling]
- [5. Input Generation]
- [6. Input Evaluation]
- [7. Configuration Updating]
- [8. Related Work]
- [9. Concluding Remarks]

## Introduction
Fuzzing: process of repeatedly running a program with generated inputs that may be syntatically or semantically malformed. <br>

This paper attempts to unifiy this field, to consolidate and distill large amount of progress in fuzzing. <br>

After introducing chosen terminology and model, this paper follows every stage of model fuzzer and present detailed overview of major fuzzers. <br>

## Systemization, Taxonomy, And Test Programs

### Fuzzing & Fuzz Testing
Fuzzing: running a **Program Under Test(PUT)** with **fuzz inputs**. <br>

Fuzz input: An input that **PUT** may not be expecting => PUT may process incorrectly and trigger unintended bahaviour

#### Defenition 1: Fuzzing
Fuzzing: Execution of the PUT using input sampled from an input space (**fuzz input space**) that protudes the expected input space of PUT. <br>

1. It will be common for fuzz input space to contain expected input space, but it is okay for former to not contain latter. <br>
2. Fuzzing normally involves a lot of repitition => "repeated executions" is quite accurate expression <br>
3. Sampling process is **not necesarilly** randomized. <br>

#### Defenition 2: Fuzz Testing
Fuzz Testing: Use of fuzzing to test if a PUT violates correctness policy. 

#### Defenition 3: Fuzzer
Fuzzer: Program that performs fuzz testing on a PUT

#### Defenition 4: Fuzz Campaign
Fuzz Campaign: Specific execution of a fuzzer on a PUT with a specific correctness policy <br>

Running PUT through fuzz campaign is to find bug that violate the specific correctness policy. <br>

Fuzz testing can actually be used to test any policy observable from execution ex: **EM-enforcable** <br>

Bug Oracle: Specific Mechanism that decides whether an execution violates the policy

#### Defenition 5: Bug Oracle
Bug Oracle: Program, perhaps as part of a fuzzer, that determines whether a given execution of the PUT violates a specific correctness policy. <br>

Although fuzz testing is focused on finding policy violation, the techniques can be diverted toward other usage. ex: PerfFuzz that reveal performance problem 

#### Definition 6: Fuzz Configuration
Fuzz configuration: Fuzz configuration of a fuzz algorithm comprises the parameter value that control fuzz algorithm. <br>

Almost all fuzz algorithm depend on some paramters beyond the PUT. Each concrete setting of the parameter is a fuzz configuration <br>

Type of values in a fuzz configuration => depend on the type of the fuzz algorithm. <br>

For example.. <br>

Fuzz algorithm that sends streams of random bytes to the PUT => simple configuration space {(PUT)}. <br>

Sophisticated fuzzers contain algorithms that accept **a set of configurations** and evolve the set over time -> includes adding, removing configurations <br>

For example, 