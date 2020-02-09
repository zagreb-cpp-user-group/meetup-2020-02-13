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

- partial support in MSVC with [Visual Studio 2019](https://devblogs.microsoft.com/cppblog/lifetime-profile-update-in-visual-studio-2019-preview-2/)
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

# Functions calls analysis

```c++
std::string_view foo( std::string const & s1, std::string const & s2 );

void myFunc() {
    std::string_view sv;
    sv = foo( "tmp1", "tmp2" );	// C: in: pset(arg1) = {__tmp1'}, pset(arg2) = {__tmp2'}
                                // ... so assume: pset(sv) = {__tmp1', __tmp2'}  [the union]
                                // ... and then at the end of this statement, __tmp1 and
                                //     __tmp2 are destroyed, so pset(sv) = {invalid}
    cout << sv[0];              // D: error: ‘s[0]’ is illegal, s was invalidated when
                                //   ‘__tmp1’ went out of scope on line C (path: C,D)
}
```
[Godbolt link](https://godbolt.org/z/wMjEHj)

---

# What if first assumption does not hold?

```c++
std::string_view foo( std::string const & s1, std::string const & ) {
    return s1;
}

void myFunc() {
    std::string_view sv;
    std::string s1 = "tmp1";
    sv = foo( s1, "tmp2" );
    std::cout << sv[0];         // false positive
}
```

[Godbolt link](https://godbolt.org/z/yWlEIM)

---

# Solution

```c++
std::string_view foo( std::string const & s1, std::string const & )
[[ gsl::post( lifetime( foo, { s1 } ) ) ]]
{
    return s1;
}

void myFunc() {
    std::string_view sv;
    std::string s1 = "tmp1";
    sv = foo( s1, "tmp2" );
    std::cout << sv[0];
}
```

---

# Limitations

- not possible to detect race conditions or global variable change - [Godbold link](https://godbolt.org/z/jtmFnL)

```c++
std::string_view foo() {
    static std::string local = some_value();
    if ( sometimes() ) local = some_other_value();
    return local;
}
void example() {
    std::string_view sv = foo();
    std::cout << sv[0];             // ok
    std::string_view sv2 = foo();
    std::cout << sv[0];             // not ok, but can't be detected
}
```

---

# Custom pointers

- using `[[ gsl::Pointer( T ) ]]` attribute

```c++
class [[ gsl::Pointer( std::string ) ]] LazyToUpper {
private:
    std::string const & originalString_;
public:
    LazyToUpper( std::string const & s ) :
        originalString_( s ) {}
    std::string uppercase() const;
};
```

[Godbold link](https://godbolt.org/z/hcq6iS)

---

# Custom owners

- using `[[ gsl::Owner( T ) ]]` attribute

```c++
template< typename T, std::size_t N >
class [[ gsl::Owner( T ) ]] SmallVector {
    [...]
};
```

---

<!-- _class: quote-large -->

# Some examples

---

# What does this program print?

```c++
template< typename ... T >
auto pack( T ... args ) {
    return std::initializer_list< int >{ args ... };
}

int main() {
    for ( auto i : pack( 1, 2, 3, 4, 5 ) ) {
        std::cout << i << ' ';
    }
    return 0;
}
```

[Godbolt link](https://godbolt.org/z/62HCW6)

---

# Classical junior developer bug - can you spot it?

```c++
#include <string>

char const * f( std::string const str ) {
    return str.c_str();
}
```

[Godbolt link](https://godbolt.org/z/ysQc5v)

---

# Substring of string_view

```c++
std::string_view substring_view( std::string_view const s, unsigned pos, unsigned len ) {
    return { s.begin() + pos, len };
}

std::string getString() { return "A long string"; }

int main() {
    auto substr = substring_view( getString(), 2, 4 );
    std::cout << substr << std::endl;
    return 0;
}
```

[Godbolt link](https://godbolt.org/z/8-rLjk)

---

# What is printed?

```c++
struct A {
    int p = 100;
};
std::unique_ptr< A > myFun() {
    std::unique_ptr< A > pa( new A() );
    return pa;
}
int main() {
    A const & rA = *myFun();
    std::cout << rA.p;
}
```
[Godbolt link](https://godbolt.org/z/SHaSPF)

---

# Can I use it?

- both clang and MSVC still have partial implementations
    - not detecing lambda captures of local variables
    - no support for lifetime pre- and post-conditions
    - no support for exception handling paths analysis
    - no support for coroutines
- Clang 10 will have this enabled by default
    - although only working checks

---

# References:

- [Herb Sutter's paper](https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf)
- [Lifetime analysis for everyone, CppCon 2019](https://youtu.be/d67kfSnhbpA)
- [Implementing the C++ Core Guidelines’ Lifetime Safety Profile in Clang, CppCon 2018](https://youtu.be/sjnp3P9x5jA)
- [Implementing the C++ Core Guidelines’ Lifetime Safety Profile in Clang, 2019 EuroLLVM Developers’ Meeting](https://youtu.be/VynWyOIb6Bk)

---

<!-- _class: quote-large -->

# Questions?

---

<!-- _class: quote-large -->

# Thank you
