---
title: Self-Registration Factory Pattern in Cpp
date: 2023-03-05 22:31:16
tags:
  - c++
  - design pattern
---

# Self-Registration Factory Pattern in C++

## Usage Scenarios

Implementing a typical factory pattern in C++ is not complicated, take the common Shape class as an example:

```c++
// ShapeFactory.cpp
#include "Shape.h"        // class Shape
#include "Triangle.h"     // class Triangle: public Shape
#include "Circle.h"       // class Circle: public Shape

std::unique_ptr<Shape> createShape(std::string_view name) {
    if (name == "triangle") return std::unique_ptr<Shape>(new Triangle());
    else if (name == "circle") return std::unique_ptr<Shape>(new Circle());
    else return nullptr;
}
```

This method is pretty straightforward and is used more, but there are two disadvantages:

1. Each concrete class implementation must be manually registered in the `ShapeFactory.cpp `. Over time, this file will be longer and there will be too many if-else branches eventually;
2. It is not easy to do isolation of function macros, and there will be nesting of preprocessor commands with poor readability after adding platform macros.

You can dynamically generate list files such as codec_list .cin FFmpeg to solve the second problem. As a cross-platform library, FFmpeg faces the very same problem of more complex functional options. Its solution is to dynamically generate this list file for as long as possible during the configuration time. Only the enabled codec will appear in the list, which looks relatively clean.

This article will introduce another way to solve the above two problems: the self-registered factory pattern.

## Implementation and Principles

Self-registration exploits the global static variable or class static member automatic initialization mechanism. In the constructor of a static variable, it will register itself into the factory method implicitly. C programming language can achieve a similar effect through `__attribute__ ((constructor)) `.

For example, the Shape example above can be written as:

*registry.h*

```cpp
template <typename ShapeType>
bool registerShape(std::string_view name) {
    ShapeFactory::instance().registerShape(name, []() {
        return std::unique_ptr<Shape>(new ShapeType());
    });
    return true;
}
```

*circle.cpp*

```cpp
class Circle: public Shape {
public:
    Circle() = default;
    double area() const override;

private:
    double m_radius = 0;
    static bool m_registered;
};

bool Circle::m_registered = registerShape<Circle>("Circle");

double Circle::area() const {
    return 3.14 * m_radius * m_radius;
}
```

`Circle::m_registered` is only used to collect the return value of the registered function and ensure that it is called. The registered function call occurs before entering the main function. After entering the main function, you can directly use the factory class to create an instance.

*main.cpp*

```cpp
int main() {
    auto shape = ShapeFactory::instance().createShape("Circle");
    if (shape)
        std::cout << "shape created" << std::endl;
    else 
        std::cout << "shape not found" << std::endl;
    return 0;
}
```

The advantage of this is obvious, since the registration of the creator takes place in the Circle source file. If we decide to disable the Circle class through the feature option, just rule out this file in the build file, no need to modify the code elsewhere, and no need to set up some functional macros to isolate the code. We don't even need to declare Circle in a header file since there is no direct reference to this class anywhere else!

This method is also highly suitable for dynamic loading plug-ins during runtime. Suppose the Circle class is designed as a plug-in, the host program only needs to invoke `dlopen ` function to load the `Circle.so`, and it will register itself into the list without any additional need for read information methods.

The sample code above can be found in different branches here:

[TsaiHao](https://github.com/TsaiHao)/**[SelfRegisterFactory](https://github.com/TsaiHao/SelfRegisterFactory)**

But this approach also inherently has some problems that are not easy to ignore. The following will introduce its shortcomings and some compromise methods.



## Problems in Practice

### Symbol Stripping of Static Linking

This is the most direct problem with the self-registration method, as the cost of implementing the code unitization above. For a static library, the linker only copies the object files directly used by the program during the linking phase. Therefore, the above `Circle.o` object file will be removed without notifying because the Circle class has no code reference other than itself!

When we **have to use static linking** without giving up the self-registration method, we can:

1. Use some compile options to force the link target to depend on self-registered symbols, in clang / gcc , this option is `-u`, see [Clang command line argument reference - Clang 17.0.0git documentation ](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-u-arg). In the above example, add an INTERFACE link option to the shape class in CMake :

   ```cmake 
   target_link_options(shape INTERFACE -u__ZN6Circle12m_registeredE)
   ```

   After that, any target links `libshape.a ` library will default rely on `Circle:: m_registerd`, thereby forcing the linker to use `Circle.o` object file.

2. Directly make the self-registered source file participate in the compilation of the upper-level target, that is, not compress `Circle.o` into the libshape.a library, but directly link the object file to the upper-level target as an additional subsidiary of the library. The implementation of CMake is:

   ```cmake
   target_sources(shape INTERFACE ${CMAKE_CURRENT_LIST_DIR}/shape/impl/Circle.cpp)
   ```

   In this way, all the targets that link the shape library will directly use `Circle.cpp` as their own source files so that the linker will not directly delete a target file such as `Circle.o`.

The above two methods are highly dependent on the compilation and CMake system. If you need cross-platform you need a more general method which I have not found yet.

### The Construction Order of Static Variables

Because the factory uses global variables to register, you must be careful not to rely on their order, and the relevant operations should be carried out after entering the main function as much as possible, for example:

1. The map that holds the creator in the Factory must be a static variable in the function scope instead of the global scope;
2. Do not create or refer to this factory in the construction of other static variables.

### Who Are Using This

Clang's plugins and various modules of clang-tidy in [Project LLVM ](https://github.com/llvm/llvm-project) use this technique:

1. Clang plugin, see [Clang Plugins - Clang 17.0.0git documentation ](https://clang.llvm.org/docs/ClangPlugins.html), dynamically loaded by dlopen at runtime, after loading, the actions in the plugin are automatically registered into the list, and there is no problem with symbol elimination.
2. Clang-tidy module registration. A new module is registered by initializing a static variable. to solve the above problem, clang-tidy declares an `extern volatile int` variable corresponding to each module in a public header file, and the definition of the variable instance is distributed in the module's source file. Because of the reference of this int variable, the corresponding module will not be removed by default.

1. 
