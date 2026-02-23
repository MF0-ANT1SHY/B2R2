F# Environment Setup Guide
==========================

This guide walks you through setting up an F# development environment on Ubuntu
with Docker installed. Choose either the native installation or Docker approach
based on your needs.

Prerequisites
-------------

- Ubuntu server (20.04 LTS or later recommended)
- Docker installed and running
- Internet connection

Method 1: Native Installation
-----------------------------

### Step 1: Install .NET SDK

F# is included with the .NET SDK. Install it using Microsoft's official
repository:

```bash
# Update package index
sudo apt-get update

# Install required dependencies
sudo apt-get install -y wget apt-transport-https software-properties-common

# Download and install Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# Update package index again
sudo apt-get update

# Install .NET SDK (includes F#)
sudo apt-get install -y dotnet-sdk-8.0
```

### Step 2: Verify Installation

Check that .NET and F# are installed correctly:

```bash
# Check .NET version
dotnet --version

# Verify F# compiler is available
which fsharpc
```

### Step 3: Create Your First F# Project

Create a simple F# console application:

```bash
# Create a new F# console app
dotnet new console -lang F# -o MyFSharpApp
cd MyFSharpApp

# Run the application
dotnet run
```

You should see `Hello from F#!` printed to the console.

### Step 4: (Optional) Install F# Interactive

For interactive development, use F# Interactive (FSI):

```bash
# Start F# Interactive
dotnet fsi

# Try some F# code in the REPL
> let x = 42;;
> printfn "The answer is %d" x;;
> #quit;;
```

Method 2: Docker-based Development
----------------------------------

If you prefer containerized development, use the official F# Docker image.

### Step 1: Pull the F# Docker Image

```bash
# Pull the latest F# SDK image
docker pull mcr.microsoft.com/dotnet/sdk:8.0
```

### Step 2: Run F# Interactive in Docker

```bash
# Start an interactive F# session
docker run -it mcr.microsoft.com/dotnet/sdk:8.0 dotnet fsi

# Or run a specific F# script
docker run -it -v $(pwd):/app mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet fsi /app/script.fsx
```

### Step 3: Create a Dockerfile for Your Project

Create a `Dockerfile` in your project directory:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0

WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet build -c Release

ENTRYPOINT ["dotnet", "run", "--project", "MyFSharpApp"]
```

Build and run your containerized F# application:

```bash
# Build the Docker image
docker build -t my-fsharp-app .

# Run the container
docker run my-fsharp-app
```

### Step 4: Development Workflow with Docker

For iterative development, mount your source code as a volume:

```bash
# Run with live code mounting
docker run -it \
  -v $(pwd):/app \
  -w /app \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet watch run
```

IDE and Editor Setup
--------------------

### Option 1: Visual Studio Code

VS Code provides excellent F# support via Ionide:

```bash
# Install VS Code (if not already installed)
sudo snap install code --classic

# Install Ionide extension for F#
code --install-extension Ionide.Ionide-fsharp
```

### Option 2: Vim/Neovim

For Vim users, install the F# language server:

```bash
# Install F# language server via npm
npm install -g @fsharp-language-server/fsharp-language-server

# Or use fsautocomplete via dotnet
dotnet tool install -g fsautocomplete
```

### Option 3: Emacs

Install `fsharp-mode` and `lsp-mode` for F# support:

```elisp
;; Add to your Emacs configuration
(use-package fsharp-mode
  :ensure t
  :hook (fsharp-mode . lsp-deferred))

(use-package lsp-mode
  :ensure t
  :commands lsp)
```

Build and Test B2R2
-------------------

Once your F# environment is ready, you can build B2R2:

```bash
# Clone the repository
git clone https://github.com/B2R2-org/B2R2.git
cd B2R2

# Restore dependencies
dotnet restore

# Build the project
dotnet build

# Run tests
dotnet test
```

Troubleshooting
---------------

### Issue: "dotnet command not found"

**Solution**: Ensure `/usr/share/dotnet` is in your PATH:

```bash
echo 'export PATH=$PATH:/usr/share/dotnet' >> ~/.bashrc
source ~/.bashrc
```

### Issue: Docker permission denied

**Solution**: Add your user to the docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Issue: F# Interactive not working

**Solution**: Use `dotnet fsi` instead of `fsharpi`:

```bash
# Modern approach
dotnet fsi

# Legacy approach (if mono is installed)
fsharpi
```

Next Steps
----------

- Read the [F# Language Guide](https://learn.microsoft.com/en-us/dotnet/fsharp/)
- Explore [B2R2 API documentation](reference/index.html)
- Join the [F# Software Foundation](https://fsharp.org/) community
- Practice with [F# for Fun and Profit](https://fsharpforfunandprofit.com/)

Additional Resources
--------------------

- [.NET Installation Guide](https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu)
- [F# Official Documentation](https://learn.microsoft.com/en-us/dotnet/fsharp/)
- [B2R2 Repository](https://github.com/B2R2-org/B2R2)
- [Ionide for VS Code](https://ionide.io/)
- [F# Language Server](https://github.com/fsharp/FsAutoComplete)
