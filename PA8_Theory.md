# PA 8: The VTable Dispatcher — Part A

**CS 379: Programming Languages (Fall 2026)**
**Author:** Ammar AlTameemy

Hierarchy used throughout:

```
class Animal { func speak() { return "..."; } func name() { return "Animal"; } }
class Dog extends Animal { func speak() { return "Woof"; } }
class Puppy extends Dog { func name() { return "Puppy"; } }
```

---

## 1. Building the Table

A VTable is an array of function pointers. Slot numbers are fixed by the
**first class that declares a method name**. A subclass starts from a copy
of its parent's array. If it overrides a method, the new pointer goes into
the **same slot index**. If it adds a new method, that gets a new slot at
the end. Here the slot order comes from `Animal`: `speak` is slot 0, `name`
is slot 1. Neither subclass adds a new method, so all three tables have
two slots.

```
            slot 0: speak                 slot 1: name
          +----------------------------+----------------------------+
Animal    | Animal.speak  -> "..."     | Animal.name   -> "Animal"  |
          |  (declared)                |  (declared)                |
          +----------------------------+----------------------------+
Dog       | Dog.speak     -> "Woof"    | Animal.name   -> "Animal"  |
          |  (OVERRIDDEN)              |  (inherited unchanged)     |
          +----------------------------+----------------------------+
Puppy     | Dog.speak     -> "Woof"    | Puppy.name    -> "Puppy"   |
          |  (inherited unchanged)     |  (OVERRIDDEN)              |
          +----------------------------+----------------------------+
```

| Class | Slot 0 `speak` | Slot 1 `name` |
|---|---|---|
| `Animal` | `Animal.speak`, declared here | `Animal.name`, declared here |
| `Dog` | `Dog.speak`, **overrides** `Animal.speak` | `Animal.name`, **inherited unchanged** from `Animal` |
| `Puppy` | `Dog.speak`, **inherited unchanged** from `Dog` | `Puppy.name`, **overrides** `Animal.name` |

What gets inherited unchanged:

- **Dog** inherits slot 1 (`name`) unchanged. `Dog.vtable[1]` is the
  *same pointer* as `Animal.vtable[1]`, not a copy of the function.
- **Puppy** inherits slot 0 (`speak`) unchanged from its direct parent
  `Dog`. So `Puppy.vtable[0]` is the same pointer as `Dog.vtable[0]`, which
  is `Dog.speak`, not `Animal.speak`. Puppy copies *Dog's* table, and Dog's
  table already has the override in it.
- Puppy does not inherit slot 1. It replaces that slot with `Puppy.name`.

This is how `build_vtable` in `dispatcher.py` works. It shallow-copies the
parent's entries, so inherited values are the same `Closure` objects. Then
it overwrites only the keys the class declares itself. A Python `dict`
keeps insertion order, so an overridden key stays at its original
position (slot index).

---

## 2. Static vs. Dynamic Dispatch

```
class Animal { func speak() { return "..."; } func name() { return "Animal"; } }
class Dog extends Animal { func speak() { return "Woof"; } }
class Puppy extends Dog { func name() { return "Puppy"; } }

Animal a = Animal();   // declared type: Animal
a = Puppy();           // runtime object: Puppy
print(a.speak());
```

- **Static dispatch** picks the method from the *declared type* of `a`,
  which is `Animal`. It would look in `Animal`'s table at compile time and
  bind the call to `Animal.speak`, so it prints **`"..."`**.
- **Dynamic dispatch** picks the method from the *object `a` holds at
  runtime*, which is a `Puppy`. It prints **`"Woof"`**.

How the dynamic call works, using the diagram from Question 1:

1. The compiler knows only that `a` is some kind of `Animal`. It knows that
   `speak` is **slot 0** in every table in this hierarchy, because
   subclasses never move a slot. So it emits "load `a`'s vtable pointer,
   read slot 0, call it."
2. At runtime, `a` points to a `Puppy` object, and that object's header
   points to **Puppy's VTable**.
3. `Puppy.vtable[0]` holds `Dog.speak`, which Puppy inherited unchanged
   from Dog. The call returns `"Woof"`, and the receiver `self` is bound to
   the Puppy object.

The static answer is wrong because the declared type only promises that
`a` holds *at least* an Animal. It says nothing about which override the
object actually carries. The real object is a Puppy, and slot 0 of the
Puppy/Dog line was overridden by Dog. Binding to `Animal.speak` would skip
that override. The program would act as if the object were a plain Animal,
which is exactly what overriding exists to prevent. In `dispatcher.py`,
`dispatch` never looks at the declared type. It always calls
`obj.vtable.resolve(name)`, and `obj.vtable` was fixed when the object was
created (`USIObject(puppy_vt, ...)`).

---

## 3. Polymorphism Trace

```
animals = [Animal(), Dog(), Puppy()];
for (x in animals) { print(x.name()); }
```

Every call site does the same thing: read the receiver's vtable pointer,
then read **slot 1** (`name`). Only the table changes from one object to
the next.

| Iteration | Element's runtime class | VTable consulted | Slot 1 (`name`) holds | Why | Output |
|---|---|---|---|---|---|
| 0 | `Animal` | `Animal.vtable` | `Animal.name` | Animal declares `name` itself | `Animal` |
| 1 | `Dog` | `Dog.vtable` | `Animal.name` | Dog does **not** override `name`. Its slot 1 was copied from Animal's table when Dog's table was built, so it falls through to Animal's code | `Animal` |
| 2 | `Puppy` | `Puppy.vtable` | `Puppy.name` | Puppy overrides `name`, so slot 1 was replaced | `Puppy` |

Final output:

```
Animal
Animal
Puppy
```

About the Dog case: the fall-through happened **when the table was
built**, not when the method was called. `dispatch` does not walk up the
class chain at runtime. It reads `Dog.vtable[1]`, and that slot already
holds the same `Animal.name` closure that `Animal.vtable[1]` holds. That is
why `dog_vt.resolve("name") is animal_vt.resolve("name")` is `True` in the
test harness. Looking up a method costs one table read no matter how deep
the inheritance chain is.

If the loop called a method that no class in the chain defines, such as
`x.fly()`, the lookup would miss in every table. `dispatch` raises
`MethodNotFoundError`, a subclass of `AttributeError`. Its message names
the class and the method, for example
`class 'Animal' has no method 'fly' (not declared or inherited)`. It never
returns `None` silently.
