---
title: Python Virtual Environments (venv) on Windows and Linux
meta_title: "Python Virtual Environments (venv) on Windows and Linux"
description: "Learn how to create and use Python virtual environments with venv
  on Windows and Linux to isolate dependencies and avoid pip conflicts."
date: ""
image: "/images/Screenshot 2026-02-04 160141.png"
categories:
  - Tutorial
  - Development
author: "David B."
tags:
  - python
  - venv
  - pip
draft: false
---
## Python Virtual Environments

If you install Python packages globally, things break sooner or later. One project needs a newer version, another needs an older one, and you end up fighting `pip` instead of writing code. Virtual environments solve this by keeping dependencies local to each project.

This post shows how to set one up and use it in a way that won’t get in your way later.

***

### Install Python and venv

Make sure Python, pip, and the venv module are installed.

#### Linux (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install python3 python3-venv python3-pip -y
```

Most systems already have Python, but `python3-venv` is sometimes missing.

#### Windows

Download and install Python from [https://www.python.org/downloads/](https://www.python.org/downloads/)

During installation:

* ✅ **Check “Add Python to PATH”**
* Leave the rest as defaults

Verify installation:

```powershell
python --version
pip --version
```

***

### Create a project directory

Keep each project in its own folder. Put the virtual environment inside it.

```bash
mkdir -p ~/my_python_project
cd ~/my_python_project
```

#### Windows equivalent

```powershell
mkdir my_python_project
cd my_python_project
```

The name doesn’t matter. The structure does.

***

### Create the virtual environment

Create a virtual environment called `venv`:

```bash
python3 -m venv venv
```

#### Windows

```powershell
python -m venv venv
```

This creates a `venv/` directory with its own Python and `pip`. Anything installed here stays here.

***

### Activate the environment

#### Linux / macOS

```bash
source venv/bin/activate
```

#### Windows (PowerShell)

```powershell
venv\Scripts\Activate.ps1
```

#### Windows (Command Prompt)

```cmd
venv\Scripts\activate.bat
```

You’ll see `(venv)` in your shell prompt. That’s how you know it’s active.

From this point on, `pip install` only affects this project.

***

### Install packages

```bash
pip install --upgrade pip
pip install flask requests
```

Install whatever your project needs. It won’t touch system Python.

If you already have a requirements file:

```bash
pip install -r requirements.txt
```

***

### Deactivate when you’re done

```bash
deactivate
```

That’s it. You’re back to the system environment.

***

### Saving dependencies

When the project is in a good state, save the package list:

```bash
pip freeze > requirements.txt
```

To rebuild the environment later:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

#### Windows rebuild

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

This works the same on another machine or server.

#### Important note (Windows only)

If PowerShell blocks activation scripts, run **once** as admin:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

***
