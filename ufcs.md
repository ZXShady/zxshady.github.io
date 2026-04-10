# I implemented UFCS inside clang 



Uniform Function Call Syntax (UFCS) is one of the most discussed features in C++.

# What is it?

In C++ we generally have two ways to perform an operation on an object

1. Member Functions: `object.method(args...)`
2. Non-member Free Functions: `function(object, args...)`

UFCS is a language feature that would allow these one syntax to be call  the other. If you wrote `x.f()`, the compiler would first look for a member function named `f`. If it didn't find one, it would look for a free function `f(x)`.

# Why is it needed?

## Generic Programming

This is one of the biggest drivers. If you are writing a template, you often don't know if the user's type uses a member function or a free function for a specific task (like `begin()` and `end()`).

today without UFCS. You have to use wrappers to cover both cases and two step dances.

```cpp
namespace fallback {
    template<typename T,int N>
    T* begin(T(&a)[N]) { return &a[0];}
    template<typename T>
    auto begin(T& t)
    requires { t.begin();}
    {
        return t.begin();
    }
}

template<typename Container>
void algo(const Container& c)
{
    // not generic enough, misses C arrays.
    auto begin1 = c.begin(); 
    
    // not generic enough, C arrays for builtin types have no associated namespace.
    auto begin2 = begin(c); 

    // generic and inline but cumbersome to write and for the compiler to optimize
    auto begin3 = [&](){ 
        if constexpr(requires {c.begin()})
            return c.begin();
        else
            return begin(c);
    }(); 

    // generic but cumbersome needs to be done everywhere
    using fallback::begin;
    auto begin4 = begin(c);
}

```

With UFCS you just use member syntax and it works for both.

```cpp
template<typename Container>
void algo(const Container& c)
{
    // done!
    auto begin = c.begin(); 
    // if c.begin() is not valid then try begin(c)
}
```

## Discoverability and autocomplete

In IDEs, when typing `object.`. triggers a list of available methods. Free functions that operate on that object (like those in `<ranges>`) don't show up in that list. UFCS would allow "C# extension methods like" making library functions much easier to find and use. especially if those functions are chainable

## Minimal and lean interfaces

Herb Sutter suggests in his awesome Guru of the week series [#84](http://www.gotw.ca/gotw/084.htm) and that if a function can be implemented using only the public interface of a class, it should be a non-member function. 
- Increases encapsulation
- Allows generic algorithms to work on any object uniformly
- It keeps class definitions small and readable instead of being 4000 lines long.
- Less dependencies in the header file class leading to faster compilation times.

However, users prefer member syntax because it feels "connected" to the object and symmetric in usage. UFCS allows the internal purity of non-member functions with the ergonomic feel of member functions.



Let's imagine a simple file api
```cpp
// myfile.hpp
namespace filesys {
class File {
public:
    File(const char* path);
    size_t length() const;
    void read(void* buf,size_t len);
private:
    void* Impl; // per os implementation
};
}
```

This header is compact and focused. It exposes only the fundamental operations, it has 0 dependencies.
But that public API is already enough to build higher-level utilities

```cpp
// myfile_utils.hpp
#include <optional> // dependency
namespace filesys {
template<typename T>
T read_object(File& file)
{
    T t;
    file.read(&t,sizeof(T));
    return t;
}

template<typename T>
std::optional<T> try_read(File& file)
{
    T t;
    // not enough bytes
    if(file.length() > sizeof(T))
        return {};
    file.read(&t,sizeof(T));
    return t;
}
}
```

With just the public API we created 2 much more convienent and powerful abstraction just but now the issue is user side.


```cpp
#include <myfile.hpp>
#include <myfile_utils.hpp> // for read_object,try_read

char buf[20];
// okay member syntax
file.read(buf,20); 

// free function syntax, have to specify namespace or rely on C++20 template ADL
Object obj = read_object<Object>(file);
std::optional<Object> obj = filesys::try_read<Object>(file); 

```


Now we broke the users flow, `read` gets the dot. `read_object` doesn't. The user deals with two different calling syntaxes for functions that conceptually belong to the same API, and must remember to qualify the namespace everytime to avoid Argument Dependant Lookup surprises. Why this distinction?

The temptation _and what many codebases actually and already do_ is to shove everything into the class.

```cpp
// myfile.hpp
#include <optional> // dependency
namespace filesys {
class File {
public:
    File(const char* path);
    size_t length() const;
    void read(void* buf,size_t len);
    template<typename T>
    T read_object();
    template<typename T>
    std::optional<T> try_read();
    
private:
    void* Impl; // per os implementation
};
}
```

```cpp
#include <myfile.hpp>
// #include <myfile_utils.hpp> does not exist anymore

char buf[20];
// uniform usage
file.read(buf,20); 
Object obj = file.read_object<Object>();
std::optional<Object> obj = file.try_read<Object>(); 
```

But this comes at a cost.

Including `<optional>` in `myfile.hpp` means every translation unit that includes this basic file header now pays for that dependency, even if it never uses `try_read` ever.

This is exactly the kind of header bloat that gives C++ its reputation for glacially slow compile times and it pushes classes toward monolithic designs.

UFCS would let us keep the clean separation:

- core operations in the class
- higher-level utilities as free functions

while still allowing users to write the operations with symmetric member function syntax.

## Arbitrary syntax tax.

In C++, the choice between member and non-member often affects syntax more than semantics.

A common defense of member functions is:

> I need access to privates.

But even that is not decisive, because a non-member can be made a friend.

```cpp
class Data {
    size_t size(){
        return m_size;
    }
    size_t m_size;
};
```

But you could just as well make it a free `friend` function, 

```cpp
class Data {
    friend size_t size(Data& self){
        return self.m_size;
    }
    size_t m_size;
};
size_t size(Data& self);

```

The real difference is not capability. It is syntax (and lookup rules).

C++ currently forces a choice between syntax and encapsulation. 
- If you want the dot, you must break encapsulation by putting the function inside the class. 
- If you want encapsulation, you must suffer the 'free function' API and non uniform usage.

UFCS decouples these two completely independent concerns giving library authors freedom to make the right engineering choice without imposing a syntax penalty on users.

## Enums/builtins with member functions

today `enum`s can't have member function.

```cpp
namespace myns {
enum class Colors {
    Green,Red,Blue
};
const char* to_string(Colors c);
}

Colors c = Colors::Red;

// unqualified opens the door to ADL surprises 
to_string(c);
// verbose
myns::to_string(c); 

// with ufcs
c.to_string(); 
// enums with members! 
// no namespace qualification needed
// IDE discoverable
```

Many APIs are object-oriented in spirit even when they are not expressed as class members (like C `FILE` API). UFCS would let the syntax reflect that.

```cpp
if (auto* f = "file.txt".std::fopen("wb"))
{
    f.fprintf("Python in my {}!".std::format("C++").c_str());
    f.fclose();
}
```

## Deduplication

Currently a significant amount class of member functions of many types are convenience functions that do not require access to private class state. UFCS provides a clear syntax for implementing such functions as non-members, reducing class API bloat and dramatically increasing reusability across types.

The classic example is the similarity between the purely const interfaces of `std::string` and `std::string_view`. `std::string_view` had to reimplement this entire interface as members, leading to significant duplication for operations that just act on *data* but their member-only implementation locks them to specific standard types.


Member Function       | Shared Overloads
----------------------|-----------------
length()              | 1
max_size()            | 1
empty()               | 1
cbegin() / cend()     | 1
crbegin() / crend()   | 1
copy()                | 1
substr()              | 1
starts_with()         | 4
ends_with()           | 4
compare()             | 9
find()                | 4
rfind()               | 4
find_first_of()       | 4
find_last_of()        | 4
find_first_not_of()   | 4
find_last_not_of()    | 4
Total                 | 46 * 2 == 92 OVERLOADS!

Thats 92 overloads! For what is essentialy algorithms already in the C++ standard library like `std::find`,`std::cbegin` and others. This would get worse as the Standard Library ends up with more types with similiar interfaces (e.g. `std::zstring_view`) more duplication will occur and this is bad for compile times and it bloats interfaces and debug builds. It also restricts usability why is `str.length()` fine but not `vector.length()`? why is `str.find('a')` fine, but not `vector.find(42)`? both are containers.

# Why isn't it here yet?

For all its appeal, UFCS is not a trivial feature at all to add to C++.

The main objections are from what I see online are.

1. unpredictable lookup complexity: C++ name lookup is already difficult
2. overload resolution ambiguity: what if both a member and a free function exist?
3. API ownership confusion and unstable APIs: `x.f()` suggests `f` belongs to the type, even if it actually comes from some unrelated namespace.
4. readability concerns: today, `x.f()` and `f(x)` may communicate different design intent which I think is bad design even without UFCS.

Most is taken from online=comments and from this [UFCS is a breaking change](https://isocpp.org/files/papers/P3027R0.html) paper.

These concerns are real. C++ is not a small language, and any feature that touches lookup and overload resolution is dangerous territory. C++ is not a playground

But I think there's a path forward.

# An idea to solve it

After thinking about the objections and reading the paper, I believe most of them stem from one design choice: making the fallback lookup too broad with ADL. If `a.foo(args...)` rewrites to `foo(a, args...)` using normal unqualified lookup scheme, you get chaos any `foo` visible at the call site could hijack the dot syntax.

So what if we constrain the lookup?

The rule: When `a.foo(args...)` fails to find an accessible (non private) member function, the compiler searches for free functions named `foo` only in the namespaces associated with the type of `a` and its base classes not at the call site, not in enclosing scopes, not via normal unqualified lookup.

Example

```cpp
namespace X {
    struct C { void member(long);};
    void member(C,int);
    void foo(const C&);
}

namespace Y {
    struct Tag {};
    void goo(X::C,Tag);
}

namespace X {
    void zoo(X::C,Y::Tag);
}
namespace Y {
    void zoo(X::C,Y::Tag);
}

void foo(X::C&);

int main() {
    X::C c;

    c.member(0); // calls X::C::member even if X::member is a better match

    Y::Tag tag;
    c.foo(); // calls X::foo, although foo(c) would find ::foo
    goo(c,tag); // finds Y::goo

    // fails, the namespaces it will find `goo`  in
    // is the associated namespaces of `c` which is X only which has no goo
    c.goo(tag);

    // okay explicit search.
    c.Y::goo(tag); 

    zoo(c,tag); // ambigious
    c.zoo(tag); // finds X::zoo
}
```
The lookup is simple, search the class for the member, then its bases for the member, then the namespaces of the class and its bases. Nothing else. No surprises. You control your type's namespace you know the functions inside it

This can be made even simpler by restricting it to the namespace of `a` only no base classes even, but I think considering base classes is logical since it is part of your API. 

This isn't a formal proposal. It's a sketch an idea that if you scope the lookup narrowly enough, most of the scary objections to UFCS dissolve and it still remains useful. The class author retains control of the API surface. The lookup is fast and predictable. Old code doesn't break. And the programmer finally gets to stop caring whether `begin` is a member or a free function and no dances anymore nor parties which if you think about it, it is not fun.

I implemented it here in this [clang fork](https://github.com/ZXShady/llvm-project/tree/ufcs) let me know what you think in the comments if you hate this or like it.


# How to use the fork

This is better in a readme but I added some compiler feature flags there are 3 "-fufcs=disabled" is the default, "-fufcs=herb" is a modified version of Herb Sutters proposal it only modifies member lookup and it has restricted name lookup search range, "-fufcs=extensions" is the same as herb except with a restriction that the function must be marked

```cpp
namespace X {
    class C{};
    void f(C&); // findable by herb
    void g(this C&); // findable by extensions, opt in
}

#if __has_feature(ufcs_all) // if herb or extensions
#if __has_feature(ufcs_herb)
#if __has_feature(ufcs_extensions)
```






## The end

Thaks for reading my article.