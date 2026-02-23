# AGENTS.md - B2R2 Binary Analysis Framework

## Project Overview

B2R2 is a binary analysis framework written in F# (.NET 10.0) maintained by SoftSec Lab @ KAIST. It supports multiple architectures including x86, ARM, MIPS, RISC-V, and more.

## Build Commands

```bash
# Restore dependencies
dotnet restore

# Build (debug)
dotnet build

# Build (release)
dotnet build -c Release

# Run all tests
dotnet test

# Run specific test project
dotnet test src/Core.Tests/B2R2.Core.Tests.fsproj

# Run specific test method
dotnet test --filter "FullyQualifiedName~BitVectorTests"

# Build without restore (faster)
dotnet build --no-restore

# Test without build (after building)
dotnet test --no-build --verbosity normal
```

## Lint Commands

```bash
# Install and run FSLint (custom linter)
git clone --depth 1 https://github.com/B2R2-org/FSLint.git
dotnet build FSLint
dotnet run --project FSLint/src/FSLint -- src/

# Or use the dotnet tool
dotnet fslint src/
```

## Code Style Guidelines

### Formatting
- **Line width**: Strictly 80 characters maximum
- **Indentation**: 2 spaces (never tabs)
- **Line endings**: Unix-style (LF) for all files
- Follow `.editorconfig` settings

### Comments
- **Documentation**: Use `///` (triple slash) above code for IntelliSense/XML docs
- **Non-documentation**: Use `(* comment *)` style

### Naming Conventions
- **Variables/parameters**: Use nouns (e.g., `address`, `instruction`)
- **Functions**: Use verbs (e.g., `parseInstruction`, `computeHash`)
- **Types**: PascalCase (e.g., `AddrRange`, `BitVector`)
- **Modules**: PascalCase

### Spacing Rules
```fsharp
// Assignment operators: spaces around =
let func = value     // Good
let func=value       // Bad

// Type annotations: space after colon
let fn (p: int) = ...     // Good
let fn (p:int) = ...      // Bad

// Tuples: space after comma
1, 2, 3              // Good
1,2,3                // Bad

// Lists/Arrays: spaces inside brackets
[ 1; 2; 3 ]          // Good
[1; 2; 3]            // Bad
[| 1; 2; 3 |]        // Good

// Generics: no spaces in brackets
func<type>           // Good
func< type >         // Bad

// Function calls: PascalCase attaches, lowercase has space
String.Replace()     // Good
String.replace ()    // Good
Func(p1, p2)         // Good for non-curried
```

### Pattern Matching
```fsharp
match x with         // Good (one space between match and with)
| Foo -> Some good
| Bar -> None

// Pipes aligned with match keyword
// Space after each pipe
// Space around -> when elements present
```

### Imports
```fsharp
namespace B2R2.ModuleName

open System
open Microsoft.VisualStudio.TestTools.UnitTesting
open B2R2
open type B2R2.BitVector   // Good for type extensions
```

### Function Definitions
- No empty lines in function bodies (refactor instead)
- One empty line between top-level bindings
- One empty line between `let rec` and `and` declarations
- Self-identifiers: use `this` when needed, otherwise `_` (avoid `__`)

### Classes/Members
```fsharp
type Class() =              // Good (space before parens)
type Class private() =      // Good (modifier attaches)

member _.Method() = value   // Good (PascalCase)
member _.method () = value  // Good (camelCase with space)
member _.Add(x, y) = ...    // Good (non-curried style)
```

### Records
```fsharp
type Record =
  { Field1: Type
    Field2: Type }          // Good

{ Field1 = value }          // Good (spaces)
```

### Array/Index Access
```fsharp
src[0] <- value             // Good
src[ 0 ] <- value           // Bad
src[1..3]                   // Good
src[ 1 .. 3 ]               // Bad
```

## Git Commit Messages

- Prepend with module tag: `[Intel]`, `[ARMv7]`, `[Build]`, `[IR]`, `[Test]`, etc.
- Subject: Max 50 chars, capitalize first letter, no period
- Body (optional): Max 72 chars per line, explains *why* not *what*
- Subject and body separated by blank line

Example:
```
[Core] Add support for 128-bit registers

Extended BitVector to handle 128-bit operations required
for new SIMD instruction support.
```

## Testing

- Uses MSTest framework
- Test classes marked with `[<TestClass>]`
- Test methods marked with `[<TestMethod>]`
- Test projects end with `.Tests.fsproj`
- Run single test: `dotnet test --filter "TestName"`

## Error Handling

- Define custom exceptions for module-specific errors
- Example: `exception RangeOverlapException`
- Document exceptions with XML comments

## Project Structure

- `src/Core/` - Core library (B2R2.Core)
- `src/BinIR/` - Binary IR (B2R2.BinIR)
- `src/FrontEnd/` - Disassemblers, parsers (per-architecture)
- `src/MiddleEnd/` - CFG, data flow, SSA
- `src/RearEnd/` - Tools (BinExplorer, BinDump, etc.)
- `src/Assembly/` - Assemblers

## Additional Notes

- Always follow existing patterns in the codebase
- See `CONTRIBUTING.md` for comprehensive style guide
- Refer to Microsoft F# style guide for general conventions
- All files must include MIT license header
