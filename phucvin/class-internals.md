# Class and Inheritance Internals in Wren

This report details the internal implementation of classes and inheritance in the Wren programming language, based on the source code in `src/vm/`.

## 1. Overview

Wren uses a class-based object system. Internally, classes are represented by `ObjClass` structures, and instances by `ObjInstance`. Inheritance is implemented via a "copy-down" mechanism where methods from the superclass are copied into the subclass's method table.

## 2. Data Structures

The core data structures are defined in `src/vm/wren_value.h`.

### `ObjClass`

The `ObjClass` struct represents a class.

```c
// src/vm/wren_value.h

struct sObjClass
{
  Obj obj;
  ObjClass* superclass;

  // The number of fields needed for an instance of this class, including all
  // of its superclass fields.
  int numFields;

  // The table of methods that are defined in or inherited by this class.
  MethodBuffer methods;

  // The name of the class.
  ObjString* name;

  // The ClassAttribute for the class, if any
  Value attributes;
};
```

*   `superclass`: Points to the parent class. This is used for `super` calls and internal checks.
*   `numFields`: Total number of fields for instances of this class. This includes fields defined in this class and all its superclasses.
*   `methods`: A dynamic array (`MethodBuffer`) storing the methods of the class. It acts as a hash table where the key is the method symbol ID.

### `ObjInstance`

The `ObjInstance` struct represents an instance of a class.

```c
// src/vm/wren_value.h

typedef struct
{
  Obj obj;
  Value fields[FLEXIBLE_ARRAY];
} ObjInstance;
```

*   `fields`: A flexible array member that stores the field values. The size of this array is determined by `classObj->numFields` at allocation time.

## 3. Class Creation

Classes are created using `wrenNewClass` in `src/vm/wren_value.c`.

```c
// src/vm/wren_value.c

ObjClass* wrenNewClass(WrenVM* vm, ObjClass* superclass, int numFields,
                       ObjString* name)
{
  // Create the metaclass.
  Value metaclassName = wrenStringFormat(vm, "@ metaclass", OBJ_VAL(name));
  wrenPushRoot(vm, AS_OBJ(metaclassName));

  ObjClass* metaclass = wrenNewSingleClass(vm, 0, AS_STRING(metaclassName));
  metaclass->obj.classObj = vm->classClass;

  wrenPopRoot(vm);

  // Make sure the metaclass isn't collected when we allocate the class.
  wrenPushRoot(vm, (Obj*)metaclass);

  // Metaclasses always inherit Class and do not parallel the non-metaclass
  // hierarchy.
  wrenBindSuperclass(vm, metaclass, vm->classClass);

  ObjClass* classObj = wrenNewSingleClass(vm, numFields, name);

  // Make sure the class isn't collected while the inherited methods are being
  // bound.
  wrenPushRoot(vm, (Obj*)classObj);

  classObj->obj.classObj = metaclass;
  wrenBindSuperclass(vm, classObj, superclass);

  wrenPopRoot(vm);
  wrenPopRoot(vm);

  return classObj;
}
```

When a new class is created:
1.  A **metaclass** is created first. The metaclass is the class of the new class object.
2.  The metaclass inherits from `Class` (`vm->classClass`).
3.  The new class object is created.
4.  The new class's `classObj` pointer is set to the metaclass.
5.  `wrenBindSuperclass` is called to set up inheritance from the user-provided `superclass`.

## 4. Inheritance

Inheritance is handled by `wrenBindSuperclass` in `src/vm/wren_value.c`.

```c
// src/vm/wren_value.c

void wrenBindSuperclass(WrenVM* vm, ObjClass* subclass, ObjClass* superclass)
{
  ASSERT(superclass != NULL, "Must have superclass.");

  subclass->superclass = superclass;

  // Include the superclass in the total number of fields.
  if (subclass->numFields != -1)
  {
    subclass->numFields += superclass->numFields;
  }
  else
  {
    ASSERT(superclass->numFields == 0,
           "A foreign class cannot inherit from a class with fields.");
  }

  // Inherit methods from its superclass.
  for (int i = 0; i < superclass->methods.count; i++)
  {
    wrenBindMethod(vm, subclass, i, superclass->methods.data[i]);
  }
}
```

Key points:
1.  **Field Inheritance**: The subclass's `numFields` is incremented by `superclass->numFields`. This ensures that `ObjInstance` has enough space for all inherited fields. The subclass's own fields are effectively appended after the superclass's fields.
2.  **Method Inheritance**: Methods are copied from the `superclass` to the `subclass`. This is a "copy-down" inheritance.
    *   It iterates through all methods in the superclass.
    *   It calls `wrenBindMethod` to add them to the subclass.
    *   Since this happens *before* the subclass defines its own methods (in the bytecode execution order), methods defined in the subclass will overwrite the copied superclass methods, achieving method overriding.

## 5. Method Binding

Methods are bound to a class using `wrenBindMethod`.

```c
// src/vm/wren_value.c

void wrenBindMethod(WrenVM* vm, ObjClass* classObj, int symbol, Method method)
{
  // Make sure the buffer is big enough to contain the symbol's index.
  if (symbol >= classObj->methods.count)
  {
    Method noMethod;
    noMethod.type = METHOD_NONE;
    wrenMethodBufferFill(vm, &classObj->methods, noMethod,
                         symbol - classObj->methods.count + 1);
  }

  classObj->methods.data[symbol] = method;
}
```

The `methods` buffer is indexed by the method's symbol ID. If the buffer isn't big enough, it's expanded. The method is then stored at the index corresponding to the symbol.

## 6. Object Instantiation

Instances are created using `wrenNewInstance` in `src/vm/wren_value.c`.

```c
// src/vm/wren_value.c

Value wrenNewInstance(WrenVM* vm, ObjClass* classObj)
{
  ObjInstance* instance = ALLOCATE_FLEX(vm, ObjInstance,
                                        Value, classObj->numFields);
  initObj(vm, &instance->obj, OBJ_INSTANCE, classObj);

  // Initialize fields to null.
  for (int i = 0; i < classObj->numFields; i++)
  {
    instance->fields[i] = NULL_VAL;
  }

  return OBJ_VAL(instance);
}
```

It allocates memory for the `ObjInstance` plus space for `classObj->numFields` values. All fields are initialized to `null`.

## 7. Method Dispatch

Method dispatch happens in the VM loop (`runInterpreter` in `src/vm/wren_vm.c`).

```c
// src/vm/wren_vm.c

    CASE_CODE(CALL_0):
    // ... other CALL opcodes ...
      // Add one for the implicit receiver argument.
      numArgs = instruction - CODE_CALL_0 + 1;
      symbol = READ_SHORT();

      // The receiver is the first argument.
      args = fiber->stackTop - numArgs;
      classObj = wrenGetClassInline(vm, args[0]);
      goto completeCall;
```

1.  **Receiver Class Lookup**: `wrenGetClassInline` retrieves the class of the receiver object.
2.  **Method Lookup**: The code jumps to `completeCall`:

```c
// src/vm/wren_vm.c

    completeCall:
      // If the class's method table doesn't include the symbol, bail.
      if (symbol >= classObj->methods.count ||
          (method = &classObj->methods.data[symbol])->type == METHOD_NONE)
      {
        methodNotFound(vm, classObj, symbol);
        RUNTIME_ERROR();
      }

      switch (method->type)
      {
          // ... dispatch based on method type ...
      }
```

The VM looks up the method in `classObj->methods` using the `symbol` index. If found, it executes the method based on its type (primitive, foreign, closure, etc.).

For `super` calls (`CODE_SUPER_X`), the dispatch logic is slightly different:

```c
// src/vm/wren_vm.c

    CASE_CODE(SUPER_0):
      // ...
      // The superclass is stored in a constant.
      classObj = AS_CLASS(fn->constants.data[READ_SHORT()]);
      goto completeCall;
```

Here, `classObj` is explicitly the superclass (stored in the constants table of the current function), bypassing the receiver's class lookup. This ensures that the method lookup starts from the superclass.

## 8. Field Access

In Wren, fields are private to the class and are accessed by index, not name. This index is determined at compile time. The compiler maps each field name to a zero-based index for each class.

There are two sets of opcodes for accessing fields: optimized access for `this` and general access.

### Accessing `this` Fields

When accessing a field on `this` (the current receiver), the compiler emits `CODE_LOAD_FIELD_THIS` or `CODE_STORE_FIELD_THIS`. These opcodes assume the receiver is already in stack slot 0 (which is always true for `this` in a method).

```c
// src/vm/wren_vm.c

    CASE_CODE(LOAD_FIELD_THIS):
    {
      uint8_t field = READ_BYTE();
      Value receiver = stackStart[0];
      ASSERT(IS_INSTANCE(receiver), "Receiver should be instance.");
      ObjInstance* instance = AS_INSTANCE(receiver);
      ASSERT(field < instance->obj.classObj->numFields, "Out of bounds field.");
      PUSH(instance->fields[field]);
      DISPATCH();
    }
```

This is highly efficient because it avoids pushing `this` onto the stack before the operation.

### General Field Access

If `this` is not in slot 0 (for example, if it's captured in a closure), the compiler emits code to load the receiver onto the stack, followed by `CODE_LOAD_FIELD` or `CODE_STORE_FIELD`.

```c
// src/vm/wren_vm.c

    CASE_CODE(LOAD_FIELD):
    {
      uint8_t field = READ_BYTE();
      Value receiver = POP();
      ASSERT(IS_INSTANCE(receiver), "Receiver should be instance.");
      ObjInstance* instance = AS_INSTANCE(receiver);
      ASSERT(field < instance->obj.classObj->numFields, "Out of bounds field.");
      PUSH(instance->fields[field]);
      DISPATCH();
    }
```

This instruction pops the receiver from the stack, verifies it's an instance, and accesses the field array using the index operand. Since fields are private, this opcode is only used when the compiler can prove that the object on the stack is indeed the instance that owns the field (e.g. `this` inside a closure).
