---
title: CMake First Project
date: 2026-05-08
categories: [Cmake]
tags: [cmake, Start]
---

# Creating Your First CMake Project

## Project Structure

```text
hello/
├── CMakeLists.txt
└── main.cpp
```

---

# Writing the Source Code

## main.cpp

```cpp
#include <iostream>

int main()
{
    std::cout << "Hello CMake Build!" << std::endl;
    return 0;
}
```

---

# Creating the CMakeLists.txt File

```bash
touch CMakeLists.txt
```
## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(Hello)
add_executable(hello main.cpp)
```

---

# Understanding the Commands

## `cmake_minimum_required()`

Defines the minimum required CMake version.

```cmake
cmake_minimum_required(VERSION 3.10)
```

---

## `project()`

Defines the project name.

```cmake
project(Hello)
```

---

## `add_executable()`

Creates an executable target.

```cmake
add_executable(hello main.cpp)
```

- `hello` → executable name
- `main.cpp` → source file

---

# Building the Project

## Step 1: Create Build Directory

---

```bash
cmake -B outputDir
```
Generate build files in a separate directory (out-of-source outputDir).
CMake reads the `CMakeLists.txt` file from the parent directory and generates platform-specific build files.

---

## Step 2: Compile the Project

```bash
cmake --build outputDir
```

---

## Step 3: Run the Program

### Linux/macOS

```bash
cd outputDir
./hello
```

### Windows
---
Go to outputDir
```powershell
hello.exe
```
---
Output:

```text
Hello from CMake!
```