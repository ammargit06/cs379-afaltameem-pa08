# PA 8: The VTable Dispatcher
## CS 379: Programming Languages (Fall 2026)

**Starter repository (GitHub template):** [cs379-pa08-vtable-dispatcher](https://github.com/cwillpinto/cs379-pa08-vtable-dispatcher). Create a copy under your own GitHub account; do not push your work to the instructor repository. The repository README explains the template and direct-clone workflows.

### 🔧 Pipeline Context
This is the **object-layout stage** of the USILang pipeline. It consumes the function-call and closure mechanism from PA 7 — every method body still executes as a call that pushes an activation record and binds parameters exactly as before — and adds a lookup layer on top: a **VTable** that maps a method name to the correct function pointer based on an object's *actual* runtime class, not the type of the variable holding it. The object/dispatch model you build here becomes what PA 9's garbage collector has to track and free next week.

---

### Part A: The Specification (50 Points)
**Instructions:** Answer the following in a separate document (PDF or Markdown).

1. **Building the Table:**
   Given a three-level single-inheritance hierarchy:
   ```
   class Animal { func speak() { return "..."; } func name() { return "Animal"; } }
   class Dog extends Animal { func speak() { return "Woof"; } }
   class Puppy extends Dog { func name() { return "Puppy"; } }
   ```
   Draw the resulting VTable **array/slot layout** for `Animal`, `Dog`, and `Puppy`. For each class, show every method slot and which class's function pointer actually occupies that slot after inheritance and overriding are applied. Be explicit about which slots `Dog` and `Puppy` inherit unchanged from their parent.

2. **Static vs. Dynamic Dispatch:**
   Write a short USILang snippet where a variable is declared/initialized as an `Animal` but assigned a `Puppy` instance at runtime, and `speak()` is called on it. State what a **statically dispatched** call would resolve to (based on the declared type) versus what a **dynamically dispatched** call resolves to (based on the actual object). Explain, using your Part A Question 1 diagram, exactly which VTable slot the dynamic call reads from and why the static answer would be wrong here.

3. **Polymorphism Trace:**
   Given a list `[Animal(), Dog(), Puppy()]`, trace the output of calling `.name()` on each element in a loop. Show, instance by instance, which class's VTable is consulted and which slot resolves — including the case where `Dog` does not override `name()` and must fall through to `Animal`'s implementation.

---

### Part B: The Implementation (50 Points)
**Instructions:** Implement the class/VTable/dispatch model in `dispatcher.py`. A resolved method follows PA 7's `Closure.call(args, caller_env)` interface. The supplied harness uses small closure-compatible method objects so you can test VTable behavior without rebuilding the PA 7 parser; your completed PA 7 closures can use the same interface.

**Requirements:**
1. Implement `VTable`:
   ```python
   class VTable:
       def __init__(self, entries: dict[str, "Closure"]):
           ...
       def resolve(self, method_name: str) -> "Closure":
           ...  # raises a clear error if the method does not exist anywhere in the chain
   ```
2. Implement the builder:
   ```python
   def build_vtable(class_node, parent_vtable: "VTable | None" = None) -> VTable:
       ...
   ```
   `build_vtable` must start from a copy of `parent_vtable`'s entries (inherited methods) and overwrite only the slots the current class actually redeclares. A subclass that doesn't override a method must resolve to the **exact same** `Closure` object its parent uses — do not duplicate or re-wrap inherited methods.
3. Implement the runtime object:
   ```python
   class USIObject:
       def __init__(self, class_descriptor, fields: dict):
           ...
   ```
   Every `USIObject` must carry a reference to its own class's `VTable` (its *actual* runtime class), regardless of what type a variable holding it was declared with.
4. Implement dispatch:
   ```python
   def dispatch(obj: "USIObject", method_name: str, args: list, caller_env) -> object:
       ...
   ```
   `dispatch` must resolve `method_name` through `obj`'s own VTable (never the declared/static type), then call `resolved_method.call([obj, *args], caller_env)`. The receiver is the first positional argument, followed by the explicit arguments. Forward the original caller environment.
5. **Correctness Requirement:** calling an undefined method on any object must raise a clear, catchable error identifying the class and method name — silent `None` returns are not acceptable.

**Verification:** Run `python test_dispatcher.py`. The harness builds the `Animal` → `Dog` → `Puppy` hierarchy from Part A, dispatches `speak()` and `name()` on instances of each class (including through a variable statically typed as a base class), and confirms every call resolves to the most-derived override while inherited-but-unoverridden methods correctly fall through the chain. If every resolution matches the expected dynamic-dispatch behavior, the script prints:
```
VTABLE_DISPATCH_VERIFIED
```

The harness also checks that dispatch passes the receiver, explicit arguments, and caller environment through the PA 7 call interface. Run it before submission; the starter's TODOs are expected to fail until you implement them.

---

### 🚀 Submission
1. `PA8_Theory.pdf` (or `.md`) — Part A answers.
2. `dispatcher.py` — your `VTable`, `build_vtable`, `USIObject`, and `dispatch` implementation.
3. Terminal output/screenshot showing the **`VTABLE_DISPATCH_VERIFIED`** Success Token from `python test_dispatcher.py`.
