# EVM Binary Analysis with B2R2

This document provides examples of how to use B2R2 to analyze Ethereum Virtual Machine (EVM) bytecode.

## Overview

B2R2 provides comprehensive support for EVM bytecode analysis including:
- Parsing and disassembling EVM instructions
- Lifting EVM instructions to LowUIR (Low-level Unified IR)
- Control Flow Graph (CFG) recovery
- Function analysis

## Getting Started

### Prerequisites

Add the required B2R2 packages to your project:

```xml
<PackageReference Include="B2R2.FrontEnd" Version="x.x.x" />
<PackageReference Include="B2R2.MiddleEnd" Version="x.x.x" />
```

### Basic Imports

```fsharp
open B2R2
open B2R2.FrontEnd
open B2R2.FrontEnd.BinLifter
open B2R2.FrontEnd.BinFile
open B2R2.MiddleEnd
open B2R2.MiddleEnd.ControlFlowAnalysis.Strategies
```

## Loading EVM Bytecode

### From Raw Bytes

```fsharp
// Create an ISA (Instruction Set Architecture) for EVM
let isa = ISA Architecture.EVM

// EVM bytecode as byte array
let bytecode = 
  [| 0x60uy; 0x80uy;    // PUSH1 0x80
     0x60uy; 0x40uy;    // PUSH1 0x40
     0x52uy;            // MSTORE
     0x60uy; 0x04uy;    // PUSH1 0x04
     0x36uy;            // CALLDATASIZE
     0x10uy;            // LT
     0x61uy; 0x00uy; 0x0euy; 0x57uy // PUSH2 0x000e JUMPI
  |]

// Create a BinHandle from raw bytes
let hdl = BinHandle(bytecode, isa)
```

### From Hex String

```fsharp
// Parse EVM bytecode from hex string
let hexString = "60806040526004361061004e57600035"
let bytes = ByteArray.ofHexString hexString
let hdl = BinHandle(bytes, isa)
```

### From File

```fsharp
// Load EVM bytecode from a file
let hdl = BinHandle("path/to/contract.bin", isa)
```

## Creating a Lifting Unit

A `LiftingUnit` provides the main interface for parsing, disassembling, and lifting instructions:

```fsharp
// Create a lifting unit from the handle
let liftingUnit = hdl.NewLiftingUnit()
```

## Parsing Instructions

### Parse a Single Instruction

```fsharp
// Parse instruction at address 0
let ins = liftingUnit.ParseInstruction(0UL)

// Access instruction properties
printfn "Address: 0x%x" ins.Address
printfn "Length: %d bytes" ins.Length
printfn "Is Branch: %b" ins.IsBranch
printfn "Is Exit: %b" ins.IsExit
```

### Parse a Basic Block

```fsharp
// Parse a basic block starting at address 0
let bblockResult = liftingUnit.ParseBBlock(0UL)

match bblockResult with
| Ok instructions ->
    for ins in instructions do
        printfn "0x%x: %s (len=%d)" 
            ins.Address 
            (ins.Disasm()) 
            ins.Length
| Error instructions ->
    printfn "Partial block parsed due to error"
```

## Disassembling EVM Code

### Disassemble Single Instruction

```fsharp
// Disassemble instruction at address 0
let disasm = liftingUnit.DisasmInstruction(0UL)
printfn "%s" disasm
// Output: PUSH1 0x80
```

### Disassemble with Options

```fsharp
// Configure to show addresses
liftingUnit.ConfigureDisassembly(showAddr = true)
let disasmWithAddr = liftingUnit.DisasmInstruction(0UL)
printfn "%s" disasmWithAddr
// Output: 0x0: PUSH1 0x80
```

## Lifting to Intermediate Representation (IR)

### Lift Single Instruction

```fsharp
// Lift instruction to LowUIR (without optimization)
let stmts = liftingUnit.LiftInstruction(0UL)

// Print IR statements
for stmt in stmts do
    printfn "%s" (stmt.ToString())
```

### Lift with Optimization

```fsharp
// Lift with optimization enabled
let optStmts = liftingUnit.LiftInstruction(0UL, optimize = true)
```

### Lift a Basic Block

```fsharp
// Lift entire basic block
let liftedBlock = liftingUnit.LiftBBlock(0UL)

match liftedBlock with
| Ok stmtArrays ->
    for stmts in stmtArrays do
        for stmt in stmts do
            printfn "%s" (stmt.ToString())
| Error stmtArrays ->
    printfn "Partial lifting due to error"
```

## Working with EVM-Specific Types

### Using the EVM Parser Directly

```fsharp
open B2R2.FrontEnd.EVM

// Create EVM parser directly
let parser = EVMParser(isa) :> IInstructionParsable

// Parse from byte span
let bytes = [| 0x60uy; 0x80uy |]  // PUSH1 0x80
let span = System.ReadOnlySpan(bytes)
let ins = parser.Parse(span, 0UL) :?> Instruction

// Access EVM-specific properties
printfn "Opcode: %A" ins.Opcode
printfn "Gas cost: %d" ins.GAS
printfn "Offset: %d" ins.Offset
```

### Working with EVM Registers

```fsharp
open B2R2.FrontEnd.EVM

// Get register factory
let regFactory = EVM.RegisterFactory() :> IRegisterFactory

// Access EVM registers
let sp = Register.toRegID Register.SP |> regFactory.GetRegVar
let pc = Register.toRegID Register.PC |> regFactory.GetRegVar
let gas = Register.toRegID Register.GAS |> regFactory.GetRegVar
```

## Control Flow Graph (CFG) Recovery

### Basic CFG Recovery

```fsharp
// Create EVM-specific CFG recovery strategy
let cfgRecovery = EVMCFGRecovery()

// Create BinaryBrew with EVM-specific configuration
let brew = EVMBinaryBrew(hdl, [| cfgRecovery |])

// Access recovered functions
for func in brew.Functions do
    printfn "Function at 0x%x" func.EntryPoint
    
    // Access CFG
    let cfg = func.CFG
    printfn "  Vertices: %d" cfg.Vertices.Count
    printfn "  Edges: %d" cfg.Edges.Count
    
    // Iterate over basic blocks
    for v in cfg.Vertices do
        printfn "  Block at 0x%x" v.VData.AddrRange.Min
```

### Advanced CFG Analysis

```fsharp
// Create custom function callback
let onFunctionIdentified addr =
    printfn "New function identified at: 0x%x" addr

// Create CFG recovery with callback
let cfgRecovery = EVMCFGRecovery(onFunctionIdentified)
let brew = EVMBinaryBrew(hdl, [| cfgRecovery |])

// Analyze a specific function
let funcAddr = 0UL
match brew.Functions.TryGetFunction(funcAddr) with
| true, func ->
    // Get the IR CFG
    let irCFG = func.IRCFG
    
    // Iterate over IR basic blocks
    for irBB in irCFG.Vertices do
        printfn "IR Block at 0x%x" irBB.VData.PPoint.Address
        for stmt in irBB.VData.Stmts do
            printfn "  %s" (stmt.ToString())
| false, _ ->
    printfn "Function not found at 0x%x" funcAddr
```

## Complete Example: Simple EVM Analyzer

```fsharp
module EVMAnalyzer

open B2R2
open B2R2.FrontEnd
open B2R2.FrontEnd.BinLifter
open B2R2.MiddleEnd
open B2R2.MiddleEnd.ControlFlowAnalysis.Strategies

let analyzeEVM (bytecode: byte[]) =
    // Create ISA for EVM
    let isa = ISA Architecture.EVM
    
    // Create binary handle
    use hdl = new BinHandle(bytecode, isa)
    
    // Create lifting unit for basic analysis
    let liftingUnit = hdl.NewLiftingUnit()
    
    printfn "=== Disassembly ==="
    let mutable addr = 0UL
    while addr < uint64 bytecode.Length do
        try
            let ins = liftingUnit.ParseInstruction(addr)
            let disasm = liftingUnit.DisasmInstruction(ins)
            printfn "0x%04x: %s" addr disasm
            addr <- addr + uint64 ins.Length
        with
        | ex ->
            printfn "0x%04x: <invalid>" addr
            addr <- addr + 1UL
    
    printfn "\n=== CFG Recovery ==="
    // Perform CFG recovery
    let cfgRecovery = EVMCFGRecovery()
    let brew = EVMBinaryBrew(hdl, [| cfgRecovery |])
    
    printfn "Recovered %d functions" brew.Functions.Count
    
    for func in brew.Functions do
        printfn "\nFunction at 0x%x" func.EntryPoint
        let cfg = func.CFG
        
        // Find jump destinations
        for v in cfg.Vertices do
            let bblock = v.VData
            printfn "  Basic Block 0x%x - 0x%x" 
                bblock.AddrRange.Min 
                bblock.AddrRange.Max
            
            // Check if block ends with a jump
            if bblock.HasLastInstruction then
                let lastIns = bblock.LastInstruction
                if lastIns.IsBranch then
                    printfn "    -> Ends with branch/jump"

// Example usage
let bytecode = ByteArray.ofHexString "60806040526004361061004e57600035"
analyzeEVM bytecode
```

## Common EVM Operations

### Working with PUSH Instructions

```fsharp
open B2R2.FrontEnd.EVM

let isa = ISA Architecture.EVM
let parser = EVMParser(isa) :> IInstructionParsable

// Parse PUSH10 instruction
let bytes = ByteArray.ofHexString "6900112233445566778899"
let span = System.ReadOnlySpan(bytes)
let ins = parser.Parse(span, 0UL) :?> Instruction

match ins.Opcode with
| PUSH10 value ->
    printfn "PUSH10 value: %A" value
| _ ->
    printfn "Not a PUSH10 instruction"
```

### Analyzing Stack Operations

```fsharp
// The lifted IR shows stack operations explicitly
let builder = ILowUIRBuilder.Default(isa, regFactory, LowUIRStream())
let stmts = ins.Translate builder

// Stack operations appear as:
// SP := SP - 32
// [SP] := value
// GAS := GAS + gas_cost
```

## Tips and Best Practices

1. **Use LiftingUnit for most operations**: It provides a high-level interface for parsing, disassembling, and lifting.

2. **Handle partial blocks**: EVM code may have invalid instructions or data sections; always check for errors when parsing basic blocks.

3. **Leverage CFG recovery**: For comprehensive analysis, use `EVMBinaryBrew` with `EVMCFGRecovery` to recover the control flow graph.

4. **Understand EVM memory model**: The lifted IR models EVM's stack, memory, and storage using appropriate abstractions.

5. **Use ByteArray helpers**: B2R2 provides `ByteArray.ofHexString` for convenient conversion from hex strings to byte arrays.

## API Reference

### Key Types

- `EVMParser`: EVM-specific instruction parser
- `EVMBinaryBrew`: EVM-specific BinaryBrew for CFG recovery
- `EVMCFGRecovery`: CFG recovery strategy for EVM
- `EVMFuncUserContext`: User context for EVM function analysis

### Key Modules

- `B2R2.FrontEnd.EVM`: Core EVM frontend (parsing, lifting)
- `B2R2.FrontEnd`: General frontend API (BinHandle, LiftingUnit)
- `B2R2.MiddleEnd.ControlFlowAnalysis.Strategies`: CFG recovery strategies

## See Also

- [B2R2 Documentation](https://github.com/B2R2-org/B2R2)
- [EVM Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [B2R2.FrontEnd.EVM README](../../src/FrontEnd/EVM/README.md)
