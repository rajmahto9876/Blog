---
title: CMake Passing Params
date: 2026-05-10
categories: [Cmake]
tags: [cmake, Passing Params]
---
# Passing Parameters from CMake to C/C++ Code

One of the powerful features of CMake is the ability to pass configuration values directly into source code during build time.

This is useful for:

- Build versioning
- Debug flags
- Feature toggles
- Platform-specific settings
- API endpoints
- Compile-time configurations

---

# Project Structure

```text
parameter-demo/
├── CMakeLists.txt
└── main.cpp
```

---

# Example 1: Passing a Simple Macro

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

project(ParameterDemo)

add_executable(app main.cpp)

target_compile_definitions(app PRIVATE
    APP_NAME="CMake_Params"
)
```

---

# main.cpp

```cpp
#include <iostream>

int main()
{
    std::cout << "Application Name: " << APP_NAME << std::endl;
    return 0;
}
```

---

# Build and Run

```bash
mkdir build
cd build

cmake ..
cmake --build .

./app
```

Output:

```text
Application Name: CMake_Params
```

---

# Understanding 
`target_compile_definitions()`

```cmake
target_compile_definitions(app PRIVATE
    APP_NAME="CMake_Params"
)
```

This command defines a compile-time macro.

Equivalent compiler flag:

```bash
-DAPP_NAME="CMake_Params"
```

---

# Example 2: Passing Version Numbers

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

project(MyApp VERSION 1.2)

add_executable(app main.cpp)

target_compile_definitions(app PRIVATE
    APP_VERSION="${PROJECT_VERSION}"
)
```

---

# main.cpp

```cpp
#include <iostream>

int main()
{
    std::cout << "App Version: " << APP_VERSION << std::endl;
    return 0;
}
```

---

# Output

```text
App Version: 1.2
```

---

# Example 3: Boolean Feature Flags

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

project(FeatureDemo)

add_executable(app main.cpp)

target_compile_definitions(app PRIVATE
    ENABLE_LOGGING
)
```

---

# main.cpp

```cpp
#include <iostream>

int main()
{

#ifdef ENABLE_LOGGING
    std::cout << "Logging Enabled" << std::endl;
#else
    std::cout << "Logging Disabled" << std::endl;
#endif

    return 0;
}
```

---

# Output

```text
Logging Enabled
```

---

# Example 4: Passing Numeric Values

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

project(BufferDemo)

add_executable(app main.cpp)

target_compile_definitions(app PRIVATE
    BUFFER_SIZE=1024
)
```

---

# main.cpp

```cpp
#include <iostream>

int main()
{

    std::cout << "Buffer Size: "
              << BUFFER_SIZE
              << std::endl;

    return 0;
}
```

---

# Output

```text
Buffer Size: 1024
```

---

# Example 5: User-Configurable Parameters

CMake allows users to pass parameters during configuration.

---

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

project(UserConfigDemo)

set(APP_MODE "Development" CACHE STRING "Application Mode")

add_executable(app main.cpp)

target_compile_definitions(app PRIVATE
    APP_MODE="${APP_MODE}"
)
```

---

# Configure Build

```bash
cmake -DAPP_MODE="Production" ..
```

---

# main.cpp

```cpp
#include <iostream>

int main()
{

    std::cout << "Mode: "
              << APP_MODE
              << std::endl;

    return 0;
}
```

---

# Output

```text
Mode: Production
```

---

# Final Thoughts

Passing parameters from CMake to C/C++ code is an essential technique for creating configurable and scalable applications.

It allows developers to:

- Customize builds easily.
- Enable/disable features.
- Inject build metadata.
- Create flexible deployment configurations.