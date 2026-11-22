# Refinery

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
int sum(int *arr, int n) {
    int result = 0;

    for (int i = 0; i < n; i++) {
        result += arr[i];
    }

    return result;
}
```

> Supporting the full native C API will be hard. Let's start with the basic stuff that is still plenty to write algorithms. We can then move onto something more versatile. This project will be most successful if we change the user story by the absolute minimal amount. This will enable AI to leverage our tool with no additional setup/instruction/skill. For example, if the entire C API is supported, we can just ask claude to write the algorithm in plain C for step one of the workflow with no additional instruction. For now, support: integer types, arrays, structs, pointers, functions, recursion, malloc.

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

For each program you prove three things:

| Theorem         | What it says                                                       |
| --------------- | ------------------------------------------------------------------ |
| Correctness     | The program computes the right result on every valid input.        |
| Exact cost      | The cost is at most an explicit formula, e.g. `C * n * (k + ℓ n)`. |
| Asymptotic cost | That formula is `O(...)`, using Mathlib's `Asymptotics.IsBigO`.    |

---

Everything after this is an AI generated draft

---

The proof process (planned):

1. Give each loop an invariant and a cost bound for each iteration.
2. A verification-condition generator splits the program into straight-line paths and
   produces one Lean goal per path.
3. Prove those goals. Calls to library routines (division, signed arithmetic and so on)
   are covered by their cost lemmas, so you never reason about how the routines work inside.

```lean
-- illustrative only; names are placeholders
theorem sum_correct ... := by
  sorry

theorem sum_cost (n k : ℕ) (σ : State) (h : ValidInput σ n k) :
    ∃ σ' c, Exec baseModel sum_prog σ σ' c ∧ c ≤ C * n * (k + ℓ n) := by
  sorry
```

Bounds are stated in terms of n (the number of items) and k (the maximum bit length of an
item), not just n. In this model, adding up n numbers of k bits is O(n · (k + log n)), not O(n).

**Unclear:** the final shape of the theorem statements, the names of the main definitions,
and how much the verification-condition generator automates.

### 5. (Optional) Re-check under a different model

The cost model is a parameter. Swap it, for example to remove division and shifts, and
re-check your bound. The C source and the shape of the proof stay the same. Only the
cost lemmas of the routines that replace removed operations change, so the bound may grow.

**Unclear:** whether this works with a one-line change or needs some proofs redone.

---

## The model

| Concept         | Choice                                                                                                                       |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Values          | Unbounded natural numbers; subtraction is truncated at 0                                                                     |
| Size of a value | ℓ(x) = bit length of x, with ℓ(0) = 1                                                                                        |
| Memory          | M : ℕ → ℕ, with indirect addressing                                                                                          |
| Base operations | `+ − × ÷ mod < =`, `and or xor`, `shl shr`, load and store, branches                                                         |
| Costs           | Roughly the bit length of the operands, e.g. `add` costs ℓ(a) + ℓ(b), `mul` costs ℓ(a)·ℓ(b), `load` costs ℓ(addr) + ℓ(value) |

C's fixed-width integers are stored as naturals below 2ʷ and wrap around through library
routines. **Unclear:** whether theorems default to this fixed-width mode or to an
unbounded mode that ignores overflow.

---

## What gets trusted

- **Checked by Lean:** everything stated about the translated program, including its
  semantics, the routine lemmas and your proofs.
- **Not checked:** clang, and the step from C source to LLVM IR. Theorems are about the
  translated IR program, and how faithfully it matches your C is assumed, not proved.
- **Undefined behaviour:** theorems assume your program has none, such as division by
  zero or signed overflow.

---

## Requirements (planned)

- Lean 4 with Mathlib
- clang, to produce LLVM IR

**Unclear:** the supported versions of each.

## Project layout

**Unclear.** It hasn't been decided yet.

## Roadmap

1. Core model and interpreter
2. Program logic (Hoare triples with costs)
3. Merge sort proved in the hand-written model language
4. Model variants
5. Model assembly and the verification-condition generator
6. Routine library
7. LLVM IR translator
8. C case studies: Dijkstra, BFS, edit distance
