# Symbols in Wren

Symbols in Wren are integers (indices) that map to strings. They are primarily used to optimize lookups for method names and variable names. Instead of comparing strings at runtime, the VM compares integer indices, which is much faster.

## What is a Symbol?

A symbol is essentially an index into a `SymbolTable`. A `SymbolTable` is a dynamic array (specifically a `StringBuffer` typedef) of `ObjString*` pointers.

*   **Header**: `src/vm/wren_utils.h`
*   **Definition**: `typedef StringBuffer SymbolTable;`
*   **Implementation**: `src/vm/wren_utils.c`

When a string is added to a `SymbolTable`, it is assigned an index. If the string already exists in the table, its existing index is returned. This process is often called "interning" in other languages.

Key functions for manipulating symbols:

*   `wrenSymbolTableEnsure(WrenVM* vm, SymbolTable* symbols, const char* name, size_t length)`: Adds a string to the table if not present, and returns its index.
*   `wrenSymbolTableFind(const SymbolTable* symbols, const char* name, size_t length)`: Returns the index of a string if found, or -1.
*   `wrenSymbolTableAdd(WrenVM* vm, SymbolTable* symbols, const char* name, size_t length)`: Adds a new string to the table and returns its index.

## Method Calling

Method calling in Wren relies heavily on symbols. The VM maintains a global symbol table for all method signatures used in the program.

### 1. The Global Method Symbol Table

The `WrenVM` struct contains a global symbol table named `methodNames`.

*   **Location**: `src/vm/wren_vm.h`
*   **Field**: `SymbolTable methodNames;` inside `struct WrenVM`.

This table stores the "signature" of every method called or defined in the program. A signature includes the method name and its arity (e.g., `count`, `add(_)`, `key=(_)`).

### 2. Compilation (Creation of Symbols)

When the compiler encounters a method call or definition, it converts the method name (and arity) into a symbol.

*   **File**: `src/vm/wren_compiler.c`
*   **Function**: `methodSymbol(Compiler* compiler, const char* name, int length)`
*   **Action**: It calls `wrenSymbolTableEnsure` on `vm->methodNames`.

```c
static int methodSymbol(Compiler* compiler, const char* name, int length)
{
  return wrenSymbolTableEnsure(compiler->parser->vm,
      &compiler->parser->vm->methodNames, name, length);
}
```

The returned integer `symbol` is then emitted into the bytecode as an operand for the call instruction.

### 3. Bytecode

The bytecode instructions for calling methods (`CODE_CALL_0` through `CODE_CALL_16`, and `CODE_SUPER_...`) take the symbol index as an argument.

*   **File**: `src/vm/wren_opcodes.h`
*   **Opcode**: `OPCODE(CALL_0, 0)` etc.

In `src/vm/wren_compiler.c`, `emitShortArg` is used to write the symbol index into the bytecode.

### 4. Runtime (Usage of Symbols)

At runtime, the interpreter reads the symbol index from the bytecode and uses it to dispatch the method.

*   **File**: `src/vm/wren_vm.c`
*   **Function**: `runInterpreter`

When executing a `CALL` instruction:

1.  It reads the symbol index: `symbol = READ_SHORT();`.
2.  It retrieves the class of the receiver.
3.  It looks up the method in the class's method table using the symbol index directly.

```c
      // If the class's method table doesn't include the symbol, bail.
      if (symbol >= classObj->methods.count ||
          (method = &classObj->methods.data[symbol])->type == METHOD_NONE)
      {
        methodNotFound(vm, classObj, symbol);
        RUNTIME_ERROR();
      }
```

**Crucial Point**: `ObjClass` has a `MethodBuffer methods`, which is a dynamic array of `Method` structures. The index in this array **corresponds exactly** to the symbol index in the global `methodNames` table.

*   **File**: `src/vm/wren_value.h`
*   **Struct**: `ObjClass` contains `MethodBuffer methods`.

### 5. Method Binding

When a class is created and methods are defined, the methods are placed into the class's `methods` array at the indices corresponding to their symbols.

*   **File**: `src/vm/wren_value.c`
*   **Function**: `wrenBindMethod(WrenVM* vm, ObjClass* classObj, int symbol, Method method)`

```c
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

This ensures that if the symbol for `toString` is `5`, then for *every* class, the implementation of `toString` (if it exists) will be at `methods.data[5]`. This allows for extremely fast O(1) method dispatch.

## Variable Symbols

Symbols are also used for module-level variables, but in a slightly different way.

### 1. Module Variable Names

Each `ObjModule` has its own symbol table for top-level variables.

*   **File**: `src/vm/wren_value.h`
*   **Struct**: `ObjModule` contains `SymbolTable variableNames` and `ValueBuffer variables`.

### 2. Compilation and Lookup

When compiling code in a module:
-   **Definitions**: `wrenDefineVariable` adds the name to `variableNames` and the value to `variables`.
-   **Lookups**: `wrenSymbolTableFind` is used to find the index of a variable name in `variableNames`.

The bytecode instructions `CODE_LOAD_MODULE_VAR` and `CODE_STORE_MODULE_VAR` use this index to access the `variables` array directly.

*   **File**: `src/vm/wren_vm.c`

```c
    CASE_CODE(LOAD_MODULE_VAR):
      PUSH(fn->module->variables.data[READ_SHORT()]);
      DISPATCH();
```

Note: Local variables do not use symbol tables at runtime; they are compiled to stack slot indices.
