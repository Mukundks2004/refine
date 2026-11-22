# Spec

## Overview

The project is a Lean 4 library with three parts:

- a formal semantics for a log-cost RAM over natural numbers (precisely what this does- in denotational terms, axiomatic, and operational)
- a script that translates ordinary C into that model, using clang and LLVM IR
- a way to prove the running time of the result

Users write normal C without needing to know the model's rules. Any C operation the model doesn't have as a primitive is replaced by a routine from a verified library, and each routine comes with a proved cost lemma. For example a verified merge sort with cost O(n log n · (k + log n)), where n is the number of keys and k is their bit length. (Comparisons cost O(k), indexing costs O(log n))

## Research Questions

- If we end up writing about this, not just using this as a tool, then we can write about these topics:

1. How does an algorithm's proven cost change when operations are added to or removed from the model?
2. Which operations does a model need before sorting can beat the Omega(n log n) comparison lower bound? We can't get a linear time comparison sort, comparisons are Theta(N log n) where n is size, k is bits of biggest number and N is total number of bits in the whole input. Does div let us get to O(N)?
3. How much cost does translating real C into the model add, and can users prove bounds using only the routines' cost lemmas, without knowing how the translation works? It would suck if translation breaks anything, so the theoretical limit is unproveable. In fact, that would derail the project. I am still not sure about this.

## Answered Questions

- Is time complexity the same regardless of RAM model? No. Reasonable models simulate each other with polynomial overhead, so classes (e.g. the `P` class) don't depend on the model. But exact bounds do. Merge sort is O(n log n) when every operation costs 1, and O(n log n · (k + log n)) in log-cost model.

- Do extra operations make merge sort faster? Not its comparison count: merge sort makes Theta(n log n) comparisons in any model. Extra operations make the overhead cheaper, and they let non-comparison algos such as radix beat n log n.

- Does the operation list change the project? Mostly no. This is really good, because Troy still hasn't confirmed the operations list for me. The semantics, translator, proof method, and routine library, all take the operation set as a parameter. Changing it changes three things: the proven bounds, which C operations need a routine instead of being primitive, and how hard some proofs are- as in, extra proof work for the user. The costs must be honest, duh. Extra reading: `Hartmanis–Simon: On the Power of Multiplication in Random Access Machines`.

- Can users write ordinary C? Yes. Turing completeness guarantees every C operation can be expressed in the model, and the routine library does it at a proven cost. Writing still won't be free of cost: i / 2 in a slimmer model becomes a routine of about O(log² i) cost, and that extra cost appears in the bound. Proofs remain manual, because working out a program's complexity automatically is undecidable in general.
  • The O(n²) target. O(n²) is true for merge sort but loose. The spec targets O(n log n · (k + log n)) instead.
  Machine model
  The machine is a log-cost RAM in the style of Cook and Reckhow, so every operation costs roughly the number of bits it touches. Charging by size stops programs from doing large amounts of work for free by building huge numbers.
  • Values. Unbounded natural numbers (ℕ). Subtraction is truncated: a ∸ b = 0 when b > a. This is the same as Lean's Nat.sub.
  • Registers. Each program declares a fixed, finite set of named registers holding values in ℕ. Because the number of registers is a constant, accessing a register adds no cost of its own.
  • Memory. A total map M : ℕ → ℕ, initially 0 everywhere. Programs access it indirectly, by loading from or storing to an address held in a register.
  • State. The registers plus memory. The structured language (see Front end) has no program counter; the model assembly (see Pipeline) adds one.
  • Size function. ℓ(x) is the bit length of x, with ℓ(0) = 1, so every operation costs at least 1. In Lean: max 1 (Nat.size x).
  • Cost of a run. The sum of the costs of the instructions executed. Termination is part of every cost theorem: a run that doesn't halt has no cost.
  \ell(x) = \max\left(1, \lceil \log_2 (x+1) \rceil\right)
  Why naturals? Naturals correspond to the unsigned integers of real machines. Sorting and array indexing never need negative numbers. Signed values can be encoded later (for example as a sign and a magnitude) at constant-factor overhead, so choosing ℕ limits nothing important and keeps the Lean proofs simpler.
  Instruction set and costs
  The base model covers the unsigned integer operations found in real instruction sets. Each is charged roughly what a simple bit-level algorithm for it would cost. This keeps translating LLVM IR direct, and it makes the original minimal set (+, ∸, ×, <, =) one variant among several (see Experiments). The cost function is a parameter of the semantics.
  Operation
  Effect
  Default cost
  const r c
  r := c
  ℓ(c)
  mov r s
  r := s
  ℓ(s)
  add r a b
  r := a + b
  ℓ(a) + ℓ(b)
  sub r a b
  r := a ∸ b (truncated)
  ℓ(a) + ℓ(b)
  mul r a b
  r := a × b
  ℓ(a) · ℓ(b)
  div r a b
  r := ⌊a / b⌋, and 0 when b = 0
  ℓ(a) · ℓ(b)
  mod r a b
  r := a mod b, and a when b = 0
  ℓ(a) · ℓ(b)
  and, or, xor r a b
  bitwise
  ℓ(a) + ℓ(b)
  shl r a k
  r := a · 2ᵏ
  ℓ(a) + k
  shr r a k
  r := ⌊a / 2ᵏ⌋
  ℓ(a) + ℓ(k)
  lt r a b
  r := 1 if a < b, else 0
  ℓ(a) + ℓ(b)
  eq r a b
  r := 1 if a = b, else 0
  ℓ(a) + ℓ(b)
  load r a
  r := M[a]
  ℓ(a) + ℓ(M[a])
  store a v
  M[a] := v
  ℓ(a) + ℓ(v)
  branch or jump
  on a register's value
  1
  Not primitive: square root, signed arithmetic, floating point and I/O. Signed operations become library routines. Floating point and I/O are rejected in v1.
  The honest-cost rule. Every primitive must cost at least the bit lengths of its operands and of its result. It must also cost no less than a simple bit-level algorithm for it needs: schoolbook for × and ÷, and linear time for addition, comparison and bitwise operations. This is why shl pays k: its result has ℓ(a) + k bits. Any operation that follows the rule changes bounds by at most a polylog factor, so the model stays realistic whichever operations are included. Unit-cost operations on unbounded numbers break the rule. With unit-cost ×, for example, repeated squaring produces exponentially large values in a few steps.
  Input conventions and size measures
  Cost theorems are stated in terms of the parameters below, never as a single unexplained n.
  Symbol
  Meaning
  n
  number of input items (for example, keys to sort)
  k
  maximum bit length of an item: k = max ℓ(key)
  N
  total input size in bits: N = Σ ℓ(key) ≤ n · k
  • Array layout. An array of length n starts at a base address b, so element i sits at address b + i. The input sits at M[0 .. n−1], and n is in a register at the start.
  • Scratch space. Algorithms use memory just past the input, for example M[n .. 2n−1] for merge sort's buffer. Every address used stays below 2n + O(1), so each address costs ℓ(address) ≤ log₂ n + O(1).
  • Output. Results go to memory locations the algorithm's specification names, for example the sorted keys back in M[0 .. n−1].
  • Reading the bounds. When k ≥ log n, the cost O(n log n · (k + log n)) simplifies to O(N log n).
  Pipeline: from C to proofs
  Users write ordinary C. clang turns it into LLVM IR, and a translator turns the IR into model assembly, the program the Lean semantics runs. Proofs are about that assembly program and use the routines' cost lemmas.

1.  Write C. The supported subset is C99 integer code: unsigned and signed integer types, arrays, structs, pointers, functions, recursion and malloc. The input follows the conventions under Input conventions.
2.  Compile with clang using a fixed set of flags (for example -O1 -fno-vectorize -fno-slp-vectorize). The result is textual LLVM IR (.ll). Fixing the flags matters, because optimisations change costs.
3.  Translate the IR into model assembly. Each IR instruction becomes either one primitive or a call to a library routine. The translator rejects floating point, vector instructions and any unknown intrinsics, with a clear error.
4.  Run or prove. #eval runs the assembly and reports its cost. Theorems state correctness and cost bounds for the C function, phrased about its translated assembly.
    There is a second way in: programs written by hand in the model language compile to the same assembly. The routine library and the early case studies use it.
    What is trusted. clang and the .ll parser are trusted. Every theorem is about the translated IR program, and how faithfully that matches the C source is assumed, not proved. CompCert's Clight is a possible later replacement if a verified front end matters.
    Model language and model assembly
    There are two languages, both defined in Lean and sharing one CostModel.
    • Model assembly is what the translator targets. It is a control-flow graph of basic blocks, with registers, primitive instructions, br and jmp, and call and ret through a stack held in memory.
    • The model language is a structured while-language for writing routines and examples by hand. It compiles to model assembly, with a proof that the compiled cost is at most a constant times the source cost.
    inductive Expr where
    | reg : Reg → Expr
    | const : ℕ → Expr
    | bin : BinOp → Expr → Expr → Expr -- add sub mul div mod and or xor shl shr lt eq
    | load : Expr → Expr -- M[e]

inductive Stmt where
| skip
| assign : Reg → Expr → Stmt
| store : Expr → Expr → Stmt -- M[e₁] := e₂
| seq : Stmt → Stmt → Stmt
| cond : Expr → Stmt → Stmt → Stmt
| loop : Expr → Stmt → Stmt -- while e ≠ 0 do s
| call : FnName → List Expr → Stmt

-- big-step semantics with cost, parameterised by the cost model
inductive Exec (cm : CostModel) : Stmt → State → State → ℕ → Prop

-- model assembly: a CFG with a small-step semantics that counts cost
structure Block where
instrs : List Instr
term : Terminator -- jmp l | br r l₁ l₂ | ret

def run (cm : CostModel) (fuel : ℕ) : Program → State → Option (State × ℕ)
• Expression cost is the cost of evaluating the operands plus the operation's cost from the table.
• Conditions are true when their value is not 0. A test costs 1 plus the cost of its expression.
• Calls push a frame onto a stack in memory. The cost of setting up the frame is charged like any other loads and stores.
Routine library
Every LLVM IR instruction becomes either a primitive or a routine written in the model language. Each routine is proved correct once and given a cost lemma, so user proofs cite lemmas such as sdiv_cost and never look inside the routine. C's w-bit integers are stored as naturals below 2ʷ. Signed values use the two's-complement reading of the same bits.
LLVM IR
Lowering in the model
Cost lemma (shape)
add, sub (iw)
primitive, then wrap: subtract 2ʷ if the result ≥ 2ʷ, add 2ʷ first if it would go below 0
O(w)
mul (iw)
primitive mul, then mod 2ʷ
O(w²)
udiv, urem
primitive div, mod
O(w²)
sdiv, srem
routine: split into sign and magnitude, divide, fix the sign
O(w²)
shl, lshr
primitive, then mod 2ʷ for shl
O(w)
ashr
routine: shr, then fill in the sign bits
O(w)
and, or, xor
primitive
O(w)
icmp unsigned (ult, eq, …)
lt, eq and combinations
O(w)
icmp signed (slt, …)
routine: compare the top bits, then the rest
O(w)
zext, trunc, sext
no-op, mod 2ʷ, routine
O(w)
load, store
primitive, multi-word values split into words
O(w + ℓ(addr))
getelementptr
address arithmetic with add and mul by a constant
O(w²)
br, switch, phi
control flow; each phi becomes moves at the ends of its predecessor blocks
O(1) per edge
call, ret, alloca
stack frame in memory, with a frame-pointer register
O(frame size · w)
malloc, free
bump allocator routine; free does nothing in v1
O(w)
Unbounded mode (optional). A translator flag treats C integers as unbounded naturals, with no wrapping. Bounds then match textbook analysis more closely, but no longer describe real C overflow behaviour. Every theorem has to say which mode it uses.
Undefined behaviour. Theorems assume the C program never triggers undefined behaviour, such as division by zero or signed overflow. The translator gives these cases fixed meanings (Lean's: division by 0 gives 0), so the semantics stay total.
Proof methodology
Each algorithm gets three theorems: a correctness theorem, an exact cost bound with explicit constants, and an asymptotic corollary.

1.  Correctness. Hoare triples {P} s {Q} over Exec. For example: the output is a permutation of the input (List.Perm) and is sorted (List.Sorted).
2.  Exact cost. A statement of the form: if Exec cm s σ σ' c and the precondition holds, then c ≤ f(n, k). The function f is an explicit closed form, such as 40 · n · ⌈log₂ n⌉ · (k + ℓ(n)) + 20.
3.  Asymptotics. f is O(g), stated with Mathlib's Asymptotics.IsBigO along Filter.atTop. Two-parameter bounds such as n and k need a product filter, or k can be fixed as a function of n.
4.  Termination. Proved together with cost: for every valid input there exist a final state σ' and a cost c such that Exec cm s σ σ' c holds.
    Loop rule. The main reusable lemma. Given a loop invariant I, a variant that decreases each iteration, and a bound bᵢ on the cost of iteration i, the loop's total cost is at most the sum of the bᵢ plus the cost of the final test. Most cost proofs then reduce to bounding a sum.
    Tooling. Build simp sets for Expr.eval and ℓ, plus lemmas such as ℓ(a + b) ≤ max(ℓ(a), ℓ(b)) + 1 and ℓ(x) ≤ ℓ(y) when x ≤ y. Check small concrete runs with #eval run before proving them.
    Proofs about translated C. Translated programs are control-flow graphs, not structured loops. So the library includes a generator for verification conditions:
5.  The user marks each loop header with an invariant and a cost bound. These are written in Lean, about the C variables, which the translator maps to registers or memory.
6.  The generator splits the program into straight-line paths between those marks.
7.  It symbolically executes each path, and produces one Lean goal per path: the invariant is preserved, and the path costs at most its share of the bound.
8.  Calls to library routines are summarised by their cost lemmas, never unfolded.
    The aim is for a user's proof to look like a textbook analysis: one invariant per loop, a sum and a big-O step.
    Case study: bottom-up merge sort
    The first verified algorithm is bottom-up merge sort. It needs only +, < and memory access, and it never halves anything: the run width starts at 1 and doubles each pass by adding it to itself.
    w := 1
    while w < n:
    i := 0
    while i < n:
    mid := min(i + w, n) -- min via lt + cond
    hi := min(i + w + w, n)
    merge M[i..mid) and M[mid..hi) into M[n+i..n+hi)
    i := i + w + w
    copy M[n..2n) back to M[0..n)
    w := w + w
    Target theorems
9.  Correctness. After the run, M[0 .. n−1] is sorted and is a permutation of the input.
10. Comparisons. At most n · ⌈log₂ n⌉ key comparisons.
11. Exact cost. At most C₁ · n · ⌈log₂ n⌉ · (k + ℓ(n)) + C₂, with C₁ and C₂ explicit.
12. Asymptotic cost. O(n log n · (k + log n)).
    Where the bound comes from. There are ⌈log₂ n⌉ passes. In each pass, every element is loaded, compared and stored a constant number of times. Each of those steps costs O(k) for the key plus O(log n) for its address and index arithmetic.
    T(n,k) = \lceil \log_2 n \rceil \cdot O\big(n\,(k + \log n)\big) = O\big(n \log n\,(k + \log n)\big)
    Experiments: changing the model
    A variant is a different CostModel: some operations removed, or costs changed. The C source and its proof stay the same. Operations that a variant removes become library routines instead, and the bound picks up the cost lemmas of those routines. This is how the project answers RQ1 and RQ2, and it tests the claim that the operation set changes the bounds but not the framework.
    Variant
    Change from base
    Question to investigate
    Base
    full table, log cost
    Reference bounds for every case study
    Minimal
    only +, ∸, ×, <, = (the original list)
    Division, shifts and bitwise operations become routines. How much do the bounds for merge sort and heap Dijkstra grow? Heap Dijkstra has to pay for computing the parent index ⌊i/2⌋.
    No × or ÷
    remove mul, div, mod
    Array indexing multiplies by a constant stride, which can be done by repeated addition. Which algorithms barely notice?
    No bitwise
    remove and, or, xor, shl, shr
    Radix sort has to extract digits with div and mod. Does it still beat n log n for small k? (RQ2)
    Cheap ×
    ×, ÷ and mod cost ℓ(a) + ℓ(b)
    Which bounds improve? This is roughly what fast multiplication algorithms achieve.
    Unit cost
    every operation costs 1
    Bounds become the textbook ones. Show concretely why unit-cost × breaks the honest-cost rule.
    Transfer lemma. Suppose model B can express each primitive of model A, either as its own primitive or as a library routine, and each costs at most c times its cost in A. Here c may be a constant or may depend on operand sizes, such as a factor of log w. Then any bound proved in A holds in B, multiplied by c. Proving this once means every variant reuses the base proofs, and only the routine cost lemmas change.
    Roadmap
    There are eight phases. Phases 1–3 build the foundation and can be done in any model. Phases 5–7 build the C pipeline. Phase 4 can run in parallel with Phases 5–7.
13. Core model. ℓ, CostModel, State, the model language, the Exec semantics, the run interpreter, and a proof that run agrees with Exec.
    ◦ Done when #eval prints the cost of a small program and the agreement theorem is proved.
14. Program logic. Hoare triples with costs, the loop rule, and simp sets for ℓ.
    ◦ Done when linear search and array sum are verified, each at O(n · (k + log n)).
15. Merge sort in the model language. All four target theorems.
    ◦ Done when the theorems are proved with no sorry.
16. Model variants. The transfer lemma, plus at least three rows of the Experiments table, run on the Phase 3 programs.
    ◦ Done when there is a short write-up of how the bounds changed under each variant, and why.
17. Model assembly. The CFG language, its cost semantics, a compiler from the model language with a constant-factor cost proof, and the verification-condition generator.
    ◦ Done when the merge sort from Phase 3, compiled to assembly, is proved with the generator.
18. Routine library. Implement and prove every routine in the Routine library table, in w-bit mode and in unbounded mode.
    ◦ Done when every routine has a correctness lemma and a cost lemma.
19. LLVM IR translator. Parse the subset of textual .ll that clang emits with the fixed flags, and emit model assembly. Reject anything unsupported.
    ◦ Done when merge sort written in plain C translates and its cost theorem is proved using only routine lemmas and the generator.
20. C case studies. Programs written in ordinary C: binary-heap Dijkstra, BFS, and one dynamic-programming algorithm such as edit distance.
    ◦ Done when each has correctness and cost theorems in the base model and in the Minimal model.
    Scope, open questions and related work
    Non-goals for v1: floating point; vector instructions; I/O; concurrency; any proof that clang compiles correctly; lower-bound proofs beyond counting comparisons; space complexity; randomized algorithms; inferring complexity automatically.
    Open questions
    [ ] Should multiplication cost ℓ(a) · ℓ(b), or should ℓ(a) + ℓ(b) be the default?
    [ ] Should theorems default to w-bit mode or unbounded mode?
    [ ] Which clang flags should be fixed? Higher optimisation levels give cleaner IR, but they make the translated program look less like the C source.
    [ ] Should the translator be written in Lean, so it can run inside proofs, or as an external tool that emits Lean terms?
    [ ] Should two-parameter bounds (n, k) be stated with a product filter, or with k as a function of n?
    Related work (cited without looking anything up; check the details before relying on them)
    • Cook and Reckhow (1973), Time Bounded Random Access Machines: the origin of the log-cost RAM.
    • Hartmanis and Simon (1974): results showing that RAMs with unit-cost multiplication are unrealistically powerful.
    • Haslbeck and Lammich: refinement with running time in Isabelle/HOL, down to LLVM, including verified sorting algorithms.
    • Charguéraud and Pottier: time credits in separation logic (CFML).
    • Nipkow et al., Functional Algorithms, Verified!: proofs about algorithms using explicit time functions.
    • Mathlib: Asymptotics.IsBigO, List.Perm, List.Sorted, Nat.size.
