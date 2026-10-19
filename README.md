# PA 8: The VTable Dispatcher

Public instructor template: https://github.com/cwillpinto/cs379-pa08-vtable-dispatcher

Full assignment: `PA_08_The_VTable_Dispatcher.md` in the template repository or
the CS379 course handout. Create your own repository from the template, clone it
locally, and push completed work only to your repository.

For a direct clone, first create an empty repository under your account, then
rename the instructor remote before adding your own:

```bash
git remote rename origin upstream
git remote add origin https://github.com/YOUR-ACCOUNT/YOUR-PA8-REPO.git
git remote -v
git push -u origin main
```

Check that `origin` points to your account. Use your actual branch name if it
is not `main`. Follow the course handout and Blackboard assignment for the
required submission items and deadline.

## Setup
Paste your own completed prior-stage files in first (`lexer.py`,
`parser.py`, `symtable.py`, `frames.py`, `parameters.py` -- only
bundled for pipeline continuity; this PA's new code doesn't call into
most of them directly).

## Run
```bash
python test_dispatcher.py
```
Complete `VTable`, `build_vtable`, `USIObject`, and `dispatch` in
`dispatcher.py`. The harness builds the Animal → Dog → Puppy hierarchy
from Part A, checks overrides, inheritance fall-through, that
unoverridden slots share the identical underlying method (not a copy),
dynamic dispatch, and error handling. Success Token: prints
`VTABLE_DISPATCH_VERIFIED`.

## Submit
1. `PA8_Theory.pdf` (or `.md`)
2. `dispatcher.py` (and your working prior-stage files)
3. Terminal output/screenshot showing `VTABLE_DISPATCH_VERIFIED`
