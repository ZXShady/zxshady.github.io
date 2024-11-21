# Understanding the Comma Operator in C++

The **comma operator** in C++ is one of those little-known gems that, when used properly, can make you look like a genius—or a complete madman, depending on your audience. In this post, we’ll explore:

- A brief history of the comma operator
- How it works and when to use it
- Why you’d want to use it
- How to troll your colleagues by overloading it

## History

The comma operator has been around since C, and like many other C operators, it was passed down to C++ like a family heirloom you didn’t ask for but now have to deal with.

## What is the function of the comma operator?

In C++, the comma operator is a **binary** operator that lets you evaluate two expressions in a single statement. It evaluates the left expression and discards it (because, who cares?), then evaluates the right expression and returns it. The syntax is straightforward:

```cpp
int main() {
    int a = 0;
    int b = 2;
    return ++a, b; // evaluates '++a', discards it, then evaluates 'b' and returns it (main returns 2)
    // a = 1
}
```

But here's something funny: the comma operator can be **chained**.

```cpp
#include <iostream>

int main() {
    int a, b, c;
    a = 0, b = 1, c = 2, std::cout << a << ' ' << b << ' ' << c;
    // prints: 0 1 2
}
```
Yea quite funny I recommend using your enter key instead.

## The For Loop Trick

Let’s say you need to increment two variables at once in a `for` loop. Sure, you could have 2 separate statements, but my enter key is broken. So, why not use the comma operator to get them both in one line? It’s like *multi-tasking* for C++.

```cpp
for (int i = 0, j = 0; i < N && j < M; ++i, ++j) {
//            ^ not a comma operator       ^ comma operator
}
```

Notice how the first comma is just separating things. It's not the comma operator—it’s **just a separator**. It’s all about context bringing us to the next point.

## Not Every Comma is an Operator

It’s easy to get confused. Not every comma you see in your code is actually the comma operator. For instance:

- When we pass parameters to functions, those commas are **just separators**.

```cpp
int calc(int a, int b, int c); 
// declarator separator, not a comma operator
//                     ^ grammar comma, not the comma operator

// when calling
calc(1, 2, 3); // still not a comma operator, just a separator
```

- The commas in template argument lists are **also not comma operators**:

```cpp
template<typename T, typename U>
// ^^^ not a comma operator, still a separator
int calcUltimate(T a, U b);

// vvv not a comma operator, still a separator
calcUltimate<int, long>(1, 2);
```

- Declaring multiple variables in one line (just don’t do it, please):

```cpp
int a, *b, &c,&&d;
```

These are **not** using the comma operator.

## Using the Comma Operator in a Comma seperator context

But wait! If you really want to use the comma operator, just throw some parentheses around it. It’s like a magic trick.

```cpp
calc((2, 1), 3); // evaluates to calc(1, 3);
//     ^ comma operator
```


## Beware of Nasal Demons

When passing arguments to functions, since the comma operator isn't always used, the **evaluation order** is **not guaranteed** and can cause **undefined behavior**.

Example:

```cpp
int calc(int a, int b);

int a;
calc(a = 1, a = 2);
// Is a == 1 or a == 2?
// Answer: It’s undefined behavior becuase you are modifying the same variable with no sequence point 
// unless you’re using C++17, then it’s implementation-defined.
// So don’t do this.
```

## Why Would I Use This?

Okay, maybe by now you’re thinking, “Why would I bother with this operator? Seems kind of useless when I have this handy enter key on my keyboard.” And, for most developers, it is. But for **template wizards**, it’s like an all-powerful spell to write 10x uglier code. Let’s dive into some cool uses that will certainly get your PR rejected.

### **Converting Expressions to `void` Without `std::void_t`**

Before C++17, you could just as easily write the `void_t` template from the standard library:

```cpp
template<typename...>
using void_t = void; // Ta-Da! 2 lines!
```

But where’s the fun in that? As template wizards, we like to write ugly, unreadable code. So, instead, we can abuse the comma operator to convert expressions to `void`:

```cpp
template<typename T, typename = void> // default case, false
struct has_addressof_operator : std::false_type {};

// using std::void_t
template<typename T>
struct has_addressof_operator<T, std::void_t<decltype(&std::declval<T>()>> : std::true_type {};

// using comma
template<typename T>
struct has_addressof_operator<T, decltype(&std::declval<T>(), void())> : std::true_type {};

```

Now it’s even more obscure, and that’s what we love.

### **Fold Expressions Since C++17**

Okay, now we're talking. The comma operator is actually **really useful** when used in fold expressions (C++17 and up). This is where the operator shines!

Let’s say you want to print all the values in a variadic template. The comma operator helps you apply an expression to all the arguments:

```cpp
template<typename... Ts>
void print_all(Ts... ts) {
    ((std::cout << ts << '\n'), ...);
}

print_all(1, 'a', "Hello World");
/* Output:
1
a
Hello World
*/
```

See? The comma operator can be your friend. It’s not all about chaos.

### **The "Poor Man's" Fold Expression (Pre-C++17)**

Not using C++17 yet? No problem! You can still use an old-school trick to simulate fold expressions with the comma operator.

```cpp
template<typename... Ts>
void print_all(Ts... ts) {
    int exprs[] = {0, (std::cout << ts << '\n', 0)...};
    // The first 0 is necessary because one may call print_all with 0 arguments and 0-sized arrays are not allowed in C++.
    // The comma operator allows us to call an expression and return a value at the same time. Here, we return 0 because it is what the array type is.
    (void)exprs; // silences unused variable warnings
}
```

## Overloading the Comma Operator

Now, let's have some fun. The comma operator is one of the many operators in C++ that can be overloaded. And, it has **really low precedence** (value 15). That means we might need to add parenthesis sometimes

```cpp
x = 1,5; // equal to (x=1),5;
x = (1,5); // equal to x = 5;
// `=` has higher precedence than `,`
```

## std::initializer_list at home
Who needs `std::initializer_list` when you have the ability to overload comma?
```cpp
std::vector<int> operator,(std::vector<int> vec, int value)
{
    vec.push_back(value);
    return vec;
}

int main() {
    std::vector<int> vec = (std::vector<int>{}, 1, 2, 3, 4, 5, 6); 
    for (int val : vec) {
        std::cout << val << '\n';
    }
    /*
    Output:
    1
    2
    3
    4
    5
    6
    */
}
```

You could also **ruin your colleague's day** by overloading the comma operator for iterators in a `for` loop. This will make them question their entire existence.

```cpp
template<typename T>
std::vector<int>::iterator& operator,(std::vector<int>::iterator&& iter, const T& /*unused*/)
{
    if (*iter % 16 == 0) // if divisible by 16
        delete &iter; // this will *probably* crash
    return iter;
}

std::vector<int> a, b;
for (auto iterA = a.begin(), iterB = b.begin(); iterA != a.end() && iterB != b.end(); iterA++, iterB++) {
    // Our overloaded comma operator in action
}
```

And when your colleague complains about the random crash in their code, just tell them to use prefix increment to teach them a lesson about prefix operators.

## Preventing the evil Overloaded Comma Operator

Want to prevent yourself from being the one who’s trolled? Easy. Just use a prvalue (pure rvalue) `void` to ensure the built-in comma operator is used.

```cpp
expr1, void(), expr2
// or
expr1, void{}, expr2
```

Since you can't overload the comma operator with a `void` argument, this trick **always** forces the use of the built-in comma operator. Phew!

## C++11 constexpr

C++11's `constexpr` was, to say the least, **really** limiting.

Here are the limitations from [cppreference.com](https://en.cppreference.com/w/cpp/language/constexpr): 
*note* all these restrictions have been lifted since C++14 so these should not be applied in new C++14 and above constexpr code

**The function body must be either deleted or defaulted or contain only the following:**
 * null statements (plain semicolons)
 * static_assert declarations
 * typedef declarations and alias declarations that do not define classes or enumerations
 * using declarations
 * using directives
 * if the function is not a constructor, **exactly one return statement**

The last point is really limiting—only 1 return statement for your entire function??

How did people do awesome things like loops, branches, and local variables? The answer: tricks, and a lot of them.

For example, to work around no statements other than return, people used commas (to do more than 1 thing) and ternaries (to simulate if statements):

```cpp
constexpr int print_and_calcCxx11(int x, bool print_result) {
    return !print_result ? x * 5: ((std::cout << "Result was " << x * 5), x * 5); 
    // can be constant-evaluated if print_result is false
    // otherwise not, since it calls std::cout 
}

// C++14 version:

constexpr int print_and_calcCxx14(int x, bool print_result) {
    int res = x * 5;
    if (print_result)
        std::cout << "Result was " << res; // not constexpr branch
    return res;
}
```

As you can see, the C++11 version is certainly a ride to read through, and quite ugly—it puts everything on a single line.


This is all I have for the comma operator it is certainly special.

Inspired by this [reddit post](https://www.reddit.com/r/cpp/comments/1g9hxyv/its_just_the_comma_operator/)