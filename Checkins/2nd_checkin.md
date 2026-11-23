# 1st Checkin

Date: TBD

## Points to Bring Up

- Fundamentally, it looks possible
- Changed bit ram to log ram
- Changed operations list since we are using log ram (div and shift allowed- check cost)
- Trying to use a full C language as the API- compile to either LLVM Ir or WASM (instruction set suits this project, maybe update to WASM)
- Go through README.md esp 'usage', which is effectively our 'failing test' (from BDD perspective)

## Notes

- Use fixed cost word ram rather than log cost
- If we are doing a 'dumb' mapping from WASM into lean model for both correctnmess and complexity bounds, then we might have the functional problemn of the definition beind really big and unwieldy
- Troy suggests cost is much easier than correctness because we can do a loop through the instruction, translate each instruction into a cost and add/multiply the costs
  - Counterpoint: even for cost this is difficult because of recursion (?)
  - So now there are two branches, the translation branch and the mathy calculating branch

## TODOs

- Research: see if anyone has done something similar (translation from assembly into something verifiable?)
- Is there literature about taking code in another language and proving it in lean
