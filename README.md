# Refine

> Nothing here is implemented yet. This describes the planned workflow.

## What Can You Do With It?

Write an algorithm in ordinary C then translate it into a formal machine model in Lean 4. You can then manually prove how long it takes to run in that model.

**The model has not been fixed yet**

The model is a log cost RAM over naturals. Each operation costs roughly the number of bits it affects.

## Usage

The planned workflow has five steps:

1. Write algorithm in C
2. Translate it
3. Run it to see its cost
4. Prove the theorem
5. Optionally, re-check the bound under a different model

## Write an Algorithm

Write normal C. You don't need to know the model's rules, because operations the model doesn't have as primitives are replaced by library routines with costs that we have already proven.

Example of simple algorithm:

```c
// sum of elements of arr with length n
int sum(int *arr, int n) {
    int result = 0;

    for (int i = 0; i < n; i++) {
        result += arr[i];
    }

    return result;
}
```

> Supporting the full native C API will be hard (IO, threads, RTOS, time operations). Let's start with the basic stuff that is still plenty to write algorithms. We can then move onto something more versatile. This project will be most successful if we change the user story by the absolute minimal amount. This will enable AI to leverage our tool with no additional setup/instruction/skill. For example, if the entire C API is supported, we can just ask claude to write the algorithm in plain C for step one of the workflow with no additional instruction. For now, support: integer types, arrays, structs, pointers, functions, recursion, malloc.

> The exact supported subset of C is whatever clang lowers to the LLVM IR instructions that the translator handles, and that list isn't final.

## Translate it

Run the translation script on your file.

```sh
translate.sh my_code.c
```

What the script does:

1. Runs `clang` with a fixed set of flags to produce LLVM IR (`my_code.ll`).
2. Translates the IR into _model assembly_ (TODO: can this be an existing assembly?). Each IR instruction becomes one model primitive or one call to a library routine.
3. Writes a Lean file containing the translated program. It should contain theorem templates (?). Unsure about things like loop invariants.

> If your program uses something unsupported, e.g., `float`, the script stops with an error naming the bad instruction

## Run it and see the cost

The generated program runs inside Lean on concrete inputs. Running it reports the final state and the total cost under the chosen cost model.

```lean
#eval run baseModel 10000 sum_prog (initState [3, 1, 4, 1, 5])
```

This step is for sanity checks, such as testing the program on small inputs or seeing how cost
grows as you change n. It doesn't prove anything.

## Write the proofs

This is the step you do by hand. Working out a program's complexity automatically is undecidable in general (rice's theorem?)

For each program you can/should prove three things. Running the script will generate the skeleton for you.

- Correctness
- Exact cost `C * n * (k + ℓ n)`
- Asymptotic cost (O(something))

Skeleton may look like:

```lean

-- Regenerated on program change (note: add something about checking for changes here)
def sum_prog : Program := xyz

-- Generated once, unless the program changes a lot
def sum_spec (specification related to state) : Prop := sorry
def sum_bound_exact (exact cost) : Prop := sorry
def sum_bound_approx (asymptotic cost) : Prop := sorry
theorem sum_correct (state) : sum_spec x y := by sorry


```

---

Everything after this is a sraft

---

## Plug and Play (TBD)

The cost model is a parameter. So we can adjust the cost model, the operations, everything should be fine.

**Unclear:** whether this works with a one line change or needs some proofs redone

---

## Notes

- Asumme no undefined behaviour
- N is input size
- n
