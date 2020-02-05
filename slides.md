---
marp: true
theme: microblink
paginate: false
---

<!-- _class: title -->

# C++ lifetime safety profile

Nenad Mikša, Head of TSI @ Microblink

@dodo at cpplang.slack.com
meetup@microblink.com

---

<!-- _class: quote-large -->

From [Microsoft](https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/):

> 70 percent of all security bugs are memory safety issues

---

# Lifetime safety profile

- [paper](https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf) written by Herb Sutter with goal of making C++ code safer
- proposes a method to detect object lifetime issues that cause memory corruption in C++
- benefits:
    - error detection happens at compile time, not at runtime
    - not part of C++ standard - compilers should issue warnings when they detect lifetime issues
    - almost no false positives, assuming modern C++ principles an practices
        - e.g. avoiding raw pointers
        - e.g. using RAII as most as possible

---

# Compiler support

- partial support in MSVC with Visual Studio 2019
- Clang implementation by Gábor Horváth and Matthias Gehre
    - available on Compiler Explorer
    - will be part of official Clang 10 release
        - however, not yet merged to trunk

---

# Pointers and Owners

- Pointer
    - refers to a resource but is not responsible for managing the resource's lifetime
    - raw pointers, `std::string_view`, `std::span`, `std::vector<T>::iterator`, ...
    - resource must outlive pointer
- Pointer detection:
    - if the type is annotated with `[[gsl::Pointer(T)]]`
    - if the type satisfies the C++ iterator requirements
    - if the type is trivially copyable, copy constructible, copy assignable, and can be dereferenced
    - if the type is a closure type of a lambda that captures its arguments by reference or captures a pointer by value
    - if the type derives from a pointer type

---

# Pointers and Owners

- Owner
    - responsible for managing resource lifetime
    - `std::string`, `std::unique_ptr`, `std::shared_ptr`, `std::vector`, ...
- Owner detection
    - if the type is annotated with `[[gsl::Owner(T)]]`
    - if the type satisfies the C++ container requirements and has a user-provided destructor
    - if the type provides a unary operator `*` and has a user-provided destructor
    - if the type derives from an owner type

---

# Lifetime tracking

- each function is analysed in isolation
- for each local variable, the analysis tracks the set of things that it points to
- [Example](https://godbolt.org/z/p-a-7m):

```c++
int* p = nullptr;   // pset(p) = {null} – records that now p is null
{
    int x = 0;
    p = &x;         // A: set pset(p) = {x} – records that now p points to x
    cout << *p;     // B: ok – *p is ok because {x} is alive
}                   // C: x destroyed => replace “x” with “invalid” in all psets
                    //                => set pset(p) = {invalid}
  cout << *p;       // D: error – because pset(p) contains {invalid}
```

---

# Generalizing pointers

```c++
string_view s;      // pset(s) = {null}
{
    char a[100];
    s=a;            // pset(s) = {a}
    cout << s[0];   // ok
}                   // x destroyed  set pset(s) = {invalid}
cout << s[0];       // error: ‘s[0]’ is illegal, s was invalidated
                    // when ‘a’ went out of scope on line C (path: A→C→D)
```

[Godbolt link](https://godbolt.org/z/3vgrnq)

---

# Generalizing owners

```c++
std::vector< int > v;
v.push_back( 5 );
auto & firstElem = v[ 0 ];  // pset(firstElem) = {v}
std::cout << firstElem;     // ok
v.push_back( 6 );           // pset(firstElem) = {invalid}
std::cout << firstElem;     // error: dereferencing a dangling pointer
```

[Godbolt link](https://godbolt.org/z/jBnKsV)

---

# Function calls

- each function analyzed in isolation
- assumptions
    - function returns values that are derived from its arguments
    - inputs are valid

---

# TODO:
- more examples
    - initializer_list
    - string_view substring
    - custom owner/pointer classes
    - capturing lambda
- references to CppCon and LLVM conference talks
- reference to paper

---

<!-- _class: quote-large -->

# Thank you