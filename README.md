

# Exception Handling in MiniML

## Introduction

### MiniML

MiniML is an implementation of an eager statically typed functional language with a compiler and abstract machine.

It has the following constructs:

- Integers with arithmetic operations `+`, `-`, and `*`.
- Since there is no exceptions defined in the language by default, there is no division operation.
- Booleans with conditional control flows and integer comparisons (`=`, `<`).
- Recursive functions and function application.
- Toplevel definitions.

### Aim of the Project

The main aim of this project is to extend the functionality of the MiniML language, and add exceptions and exception handling.

---

## Features

The language now supports integer division.

Additionally, two new constructs have been added to the language:

- `raise`: Used to raise exceptions.
- `try-with`: Used to handle exceptions.

The exceptions considered are:

- `DivisionByZero`: Raised by the machine when the divisor in a division operation is zero.
- `GenericException of int`: A Generic Exception type that can be used to define custom exceptions. Takes an integer code to represent the exception type.

Also, type checking has been modified so that incorrect datatypes are handled at runtime rather than before execution. To implement this, the definition of closures has also been modified.

---

## Type System

The type checking system has been changed to accommodate runtime incorrect datatype handling. The key changes:

- Added a new `exptn` type in syntax for exceptions:
  ```ocaml
  type exptn =
    | DivisionByZero
    | GenericException of int
  ```
- Consequently, syntax type `TExptn` and machine value type `MExptn` have also been added.
- In `type_check.ml`, bypassing the incorrect datatype error is now possible:
  ```ocaml
  | Plus (a, b) ->
      (try check ctx TInt a; check ctx TInt b; TInt with Type_Error -> TExptn)
  ```
- Example in `machine.ml`:
  ```ocaml
  let add = function
    | (MInt x) :: (MInt y) :: s -> MInt (y + x) :: s
    | _ -> [MExptn (Syntax.GenericException (-1))]
  ```
- `MClosure` has been modified to:
  ```ocaml
  MClosure of name * frame * environ * Syntax.ty
  ```
  The parameter's expected type is checked at runtime.
- For `try-with`, type checking works like:
  - If test expression is not of type `TExptn`, return its type.
  - Else, return the type of the last case expression in `with`.

> Note: Division by zero still returns type `TInt` as the divisor value isn't available during type checking.

---

## Exception Handling

### Exceptions

Exceptions that can be thrown by the machine:

- `DivisionByZero`: when dividing by zero (e.g. `5 / 0`).
- `GenericException -1`: for incorrect operand types (e.g. `5 + true`, `false < 4`).
- `GenericException 1`: for incorrect function parameter types (e.g. `fact true`).

Detection mechanisms:

- **DivisionByZero**:
  ```ocaml
  let div = function
    | (MInt 0) :: (MInt _) :: s -> MExptn Syntax.DivisionByZero :: s
    | (MInt x) :: (MInt y) :: s -> MInt (y / x) :: s
    | _ -> [MExptn (Syntax.GenericException (-1))]
  ```
- **GenericException -1**: Thrown for incorrect operand types.
- **GenericException 1**: 
  - `MClosure` modified to store expected param type.
  - Runtime type is matched using helper `get_type`.
  
Manual raising:
```ocaml
raise DivisionByZero;;
```

### Exception Handling

Using `try-with` syntax:
```ocaml
try { test-expression } with { | error1 -> exp1 | error2 -> exp2 ... }
```

Parsed in `parser.mly` as:
```ocaml
TRY e = expr WITH LBRACE cases = nonempty_list(case) RBRACE
  { Try (e, cases) }

case:
  | PIPE e=exptn TARROW e1=expr { (e, e1) }
```

Type checking logic:
```ocaml
| Try (e, cases) ->
    let ty = type_of ctx e in
    let rec match_cases cases exp_case = match cases with
      | (_, exp) :: body -> let t' = type_of ctx exp in match_cases body t'
      | [] -> exp_case
    in
    let t' = match_cases cases ty in
    if ty = TExptn then t' else ty
```

**Limitations**:
- DivisionByZero is not `TExptn` so its `with` block must return `int`.
- If `with` block has mixed types, result type is that of last case.

Machine-level implementation:
- After evaluation, if result is an exception, match against `with` cases.
- If matched, run the associated expression.

---

## Example Cases

### Exceptions Raised

- **Division by zero**:
  ```ocaml
  miniML> 1/0;;
  - : int = Division By Zero
  ```
- **Wrong operands**:
  ```ocaml
  miniML> 1+true;;
  - : error = Generic Exception -1
  ```
- **Wrong function parameter**:
  ```ocaml
  miniML> let double = fun f (n:int) : int is 2*n;;
  double : int -> int = <fun>
  miniML> double true;;
  - : error = Generic Exception 1
  ```
- **Manual raise**:
  ```ocaml
  miniML> raise GenericException 42;;
  - : error = Generic Exception 42
  ```

### `try-with` Usage

- **Simple handling**:
  ```ocaml
  miniML> try {21/0} with {|DivisionByZero -> 0};;
  - : int = 0
  ```
- **Multiple cases**:
  ```ocaml
  miniML> try {10 + false} with {|DivisionByZero -> 10 | GenericException -1 -> 20};;
  - : int = 20
  ```
- **Unhandled exception**:
  ```ocaml
  miniML> try {3/0} with {|GenericException 1 -> 9};;
  - : int = Division By Zero
  ```
- **Nested blocks**:
  ```ocaml
  miniML> try {4 + try{4/0} with {|DivisionByZero -> false}} with {|GenericException -1 -> 140};;
  - : int = 140
  ```

---

## Limitations & Issues

- **Invalid raise argument**:
  ```ocaml
  miniML> raise 5;;
  Syntax error...
  ```
- **Duplicate exception cases**:
  ```ocaml
  miniML> try {2/0} with {|DivisionByZero -> 0 | DivisionByZero -> 5};;
  - : int = 0
  ```
- **Mixed case types**:
  ```ocaml
  miniML> try {3+true} with {|GenericException -1 -> 5 | DivisionByZero -> false};;
  - : bool = 5
  ```
- **Fixed return type**:
  ```ocaml
  miniML> try {5/0} with {|DivisionByZero -> false};;
  - : int = false
  ```
- **Manual raise unhandled**:
  ```ocaml
  miniML> try {raise DivisionByZero} with {|DivisionByZero -> 0};;
  - : int = Division By Zero
  ```

---

## Challenges

- Understanding parser structure.
- Defining a new type `exptn`.
- Introducing new syntax constructs.
- Bypassing type-checking to support runtime errors.

---

## How to Use

1. Install **OCaml** and set up the **OCaml Development Environment**.
2. Activate the **opam switch**.
3. Clean previous build:
   ```bash
   dune clean
   ```
4. Rebuild the project:
   ```bash
   dune build src/miniml
   ```
5. Run the REPL or executable as required.
