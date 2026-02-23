# Docker-based EVM Binary Analysis with B2R2

This guide walks you through setting up a Docker-based F# project from scratch to analyze Ethereum Virtual Machine (EVM) bytecode using B2R2.

## Overview

This tutorial covers:
- Setting up a Docker-based F# development environment
- Creating a new F# project from scratch
- Adding B2R2 dependencies
- Writing EVM analysis code
- Running the analysis on EVM bytecode

## Prerequisites

- Docker installed and running
- Basic knowledge of F# syntax
- EVM bytecode to analyze (hex string or binary file)

## Step 1: Create Project Directory Structure

First, create a directory for your project:

```bash
mkdir -p ~/evm-analyzer/src
cd ~/evm-analyzer
```

## Step 2: Create a Dockerfile

Create a `Dockerfile` that sets up the F# environment with B2R2 dependencies:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0

WORKDIR /app

# Install any additional tools if needed
RUN apt-get update && apt-get install -y \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copy project files
COPY . .

# Restore dependencies
RUN dotnet restore

# Build the project
RUN dotnet build

# Default command
CMD ["dotnet", "run"]
```

## Step 3: Initialize the F# Project

Create a new F# console project within the Docker environment:

```bash
# Create a new F# console application
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet new console -lang F# -o src -n EVMAnalyzer
```

This creates:
- `src/EVMAnalyzer.fsproj` - Project file
- `src/Program.fs` - Main program file
- `src/obj/` - Build artifacts

## Step 4: Add B2R2 Dependencies

Add the required B2R2 NuGet packages to your project:

```bash
# Add B2R2.FrontEnd package
docker run -it --rm \
  -v $(pwd):/app \
  -w /app/src \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet add package B2R2.FrontEnd

# Add B2R2.MiddleEnd package
docker run -it --rm \
  -v $(pwd):/app \
  -w /app/src \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet add package B2R2.MiddleEnd
```

## Step 5: Write the EVM Analyzer Code

Replace the contents of `src/Program.fs` with the following complete EVM analyzer:

```fsharp
/// EVM Binary Analyzer using B2R2
/// This tool analyzes Ethereum Virtual Machine bytecode

module EVMAnalyzer

open System
open B2R2
open B2R2.FrontEnd
open B2R2.FrontEnd.BinLifter
open B2R2.MiddleEnd
open B2R2.MiddleEnd.ControlFlowAnalysis.Strategies

/// Analyzes EVM bytecode and prints disassembly
let disassembleEVM (bytecode: byte[]) =
    printfn "=== Disassembly ==="
    let isa = ISA Architecture.EVM
    use hdl = new BinHandle(bytecode, isa)
    let liftingUnit = hdl.NewLiftingUnit()
    
    let mutable addr = 0UL
    while addr < uint64 bytecode.Length do
        try
            let ins = liftingUnit.ParseInstruction(addr)
            let disasm = liftingUnit.DisasmInstruction(ins)
            printfn "0x%04x: %s (len=%d)" addr disasm ins.Length
            addr <- addr + uint64 ins.Length
        with
        | ex ->
            printfn "0x%04x: <invalid or data>" addr
            addr <- addr + 1UL

/// Performs CFG recovery and analysis
let analyzeCFG (bytecode: byte[]) =
    printfn "\n=== CFG Analysis ==="
    let isa = ISA Architecture.EVM
    use hdl = new BinHandle(bytecode, isa)
    
    let cfgRecovery = EVMCFGRecovery()
    let brew = EVMBinaryBrew(hdl, [| cfgRecovery |])
    
    printfn "Recovered %d functions" brew.Functions.Count
    
    for func in brew.Functions do
        printfn "\nFunction at 0x%x" func.EntryPoint
        let cfg = func.CFG
        printfn "  Vertices: %d" cfg.Vertices.Count
        printfn "  Edges: %d" cfg.Edges.Count
        
        for v in cfg.Vertices do
            let bblock = v.VData
            printfn "  Basic Block 0x%x - 0x%x" 
                bblock.AddrRange.Min 
                bblock.AddrRange.Max
            
            if bblock.HasLastInstruction then
                let lastIns = bblock.LastInstruction
                if lastIns.IsBranch then
                    printfn "    -> Ends with branch/jump instruction"

/// Lifts EVM bytecode to LowUIR (Intermediate Representation)
let liftToIR (bytecode: byte[]) =
    printfn "\n=== Lifting to IR ==="
    let isa = ISA Architecture.EVM
    use hdl = new BinHandle(bytecode, isa)
    let liftingUnit = hdl.NewLiftingUnit()
    
    let mutable addr = 0UL
    let mutable count = 0
    while addr < uint64 bytecode.Length && count < 10 do
        try
            let stmts = liftingUnit.LiftInstruction(addr)
            printfn "\n0x%04x:" addr
            for stmt in stmts do
                printfn "  %s" (stmt.ToString())
            
            let ins = liftingUnit.ParseInstruction(addr)
            addr <- addr + uint64 ins.Length
            count <- count + 1
        with
        | ex ->
            addr <- addr + 1UL

/// Main entry point
[<EntryPoint>]
let main argv =
    printfn "B2R2 EVM Binary Analyzer"
    printfn "========================\n"
    
    // Example EVM bytecode (simple contract)
    let exampleBytecode = 
        ByteArray.ofHexString "60806040526004361061004e57600035"
    
    // Check if bytecode was provided as argument
    let bytecode =
        if argv.Length > 0 then
            try
                ByteArray.ofHexString argv[0]
            with
            | ex ->
                printfn "Warning: Failed to parse hex string, using example"
                exampleBytecode
        else
            printfn "Usage: dotnet run <hex-string>"
            printfn "Using example bytecode...\n"
            exampleBytecode
    
    printfn "Analyzing %d bytes of EVM bytecode\n" bytecode.Length
    
    // Run analyses
    disassembleEVM bytecode
    analyzeCFG bytecode
    liftToIR bytecode
    
    printfn "\nAnalysis complete!"
    0
```

## Step 6: Create a Docker Compose File (Optional)

For easier management, create a `docker-compose.yml`:

```yaml
version: '3.8'

services:
  evm-analyzer:
    build: .
    volumes:
      - ./src:/app
      - ./contracts:/contracts:ro
    working_dir: /app
    command: dotnet run
```

## Step 7: Build and Run the Project

### Build the Docker image:

```bash
docker build -t evm-analyzer .
```

### Run with example bytecode:

```bash
docker run --rm evm-analyzer
```

### Run with your own EVM bytecode:

```bash
# Pass bytecode as hex string
docker run --rm evm-analyzer dotnet run "6080604052348015610010576000fd5b50"

# Or mount a file containing bytecode
docker run --rm \
  -v $(pwd)/contracts:/contracts \
  evm-analyzer dotnet run "$(cat contracts/contract.hex)"
```

### Run with live development mode:

```bash
# Mount source code and use watch mode for auto-rebuild
docker run -it --rm \
  -v $(pwd)/src:/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet watch run
```

## Step 8: Analyze an EVM Contract File

If you have an EVM binary file (e.g., `contract.bin`):

```fsharp
// Add this function to Program.fs

/// Load bytecode from file
let loadBytecodeFromFile (filepath: string) =
    try
        IO.File.ReadAllBytes(filepath)
    with
    | ex ->
        printfn "Error loading file: %s" ex.Message
        [||]

// Update main function to support file input
let main argv =
    // ... existing code ...
    
    let bytecode =
        if argv.Length > 0 then
            if argv[0].EndsWith(".bin") then
                loadBytecodeFromFile argv[0]
            else
                ByteArray.ofHexString argv[0]
        else
            exampleBytecode
    // ... rest of main ...
```

Run with a file:

```bash
docker run --rm \
  -v $(pwd)/contracts:/contracts \
  evm-analyzer dotnet run /contracts/contract.bin
```

## Project Structure

Your project should now look like this:

```
evm-analyzer/
├── Dockerfile
├── docker-compose.yml (optional)
└── src/
    ├── EVMAnalyzer.fsproj
    ├── Program.fs
    └── obj/
        └── ...
```

## Advanced Usage

### Custom Analysis Script

Create a standalone analysis script `analyze.fsx`:

```fsharp
#!/usr/bin/env dotnet fsi

#r "nuget: B2R2.FrontEnd"
#r "nuget: B2R2.MiddleEnd"

open B2R2
open B2R2.FrontEnd
open B2R2.MiddleEnd
open B2R2.MiddleEnd.ControlFlowAnalysis.Strategies

let analyze (hexString: string) =
    let bytecode = ByteArray.ofHexString hexString
    let isa = ISA Architecture.EVM
    use hdl = new BinHandle(bytecode, isa)
    let liftingUnit = hdl.NewLiftingUnit()
    
    // Parse and print each instruction
    let mutable addr = 0UL
    while addr < uint64 bytecode.Length do
        let ins = liftingUnit.ParseInstruction(addr)
        printfn "%s" (liftingUnit.DisasmInstruction(ins))
        addr <- addr + uint64 ins.Length

// Run analysis
if fsi.CommandLineArgs.Length > 1 then
    analyze fsi.CommandLineArgs[1]
```

Run the script:

```bash
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet fsi analyze.fsx "6080604052"
```

### Interactive Development

Start an interactive F# session with B2R2 loaded:

```bash
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet fsi

# In FSI, reference packages:
> #r "nuget: B2R2.FrontEnd";;
> #r "nuget: B2R2.MiddleEnd";;
> open B2R2;;
> let isa = ISA Architecture.EVM;;
```

## Troubleshooting

### Issue: Package restore fails

**Solution**: Ensure you have internet connectivity in the container:

```bash
docker run --rm mcr.microsoft.com/dotnet/sdk:8.0 ping -c 1 nuget.org
```

### Issue: Permission denied when mounting volumes

**Solution**: Run container with your user ID:

```bash
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  --user $(id -u):$(id -g) \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet run
```

### Issue: "Could not load file or assembly"

**Solution**: Rebuild the project:

```bash
docker run --rm \
  -v $(pwd)/src:/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet clean && dotnet restore && dotnet build
```

## Next Steps

1. **Extend the analyzer**: Add more sophisticated analysis like:
   - Function signature detection
   - Control flow analysis
   - Vulnerability detection patterns

2. **Export results**: Modify the code to output analysis results to JSON or other formats

3. **Batch processing**: Create a script to analyze multiple contracts

4. **Explore B2R2 API**: Refer to the [EVM Example Documentation](EVM_example.md) for more advanced features

## Resources

- [B2R2 GitHub Repository](https://github.com/B2R2-org/B2R2)
- [EVM Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [F# Language Guide](https://learn.microsoft.com/en-us/dotnet/fsharp/)
- [.NET Docker Images](https://hub.docker.com/_/microsoft-dotnet-sdk/)
