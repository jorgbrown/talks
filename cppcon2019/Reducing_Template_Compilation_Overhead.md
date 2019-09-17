name: title-layout
layout: true
class: center, middle, title

---
name: basic-layout
layout: true
class: left, top

---
name: title
template: title-layout

# Reducing Template Compilation Overhead
.footnote[Jorg Brown, <jorg.brown@gmail.com>, [@jorgbrown](https://twitter.com/jorgbrown)]

???
Intro:
- I'm Jorg Brown
- I work with Chandler Carruth on Google's C++ PL platform
- Obsessed with performance and optimization.

I've wanted to have given a talk at CppCon for some time, and finally have one.

---
name: Background
class: middle, center

![First Guess](templates_memory_bender_meme.jpg)

???

---
name: Background
class: left, middle

# Reducing Template Overhead
## First, some background
1. Hard to measure when things are going well
1. BubbleSort is better than you think.
1. Compile-time calculation can be surprising

???
Well-designed O(n) template code is in the noise, as far as overhead goes.
So is O(n^2) overhead, typically.
BubbleSort isn't just OK for low N; it's OPTIMAL up to N=3.

---
### It's hard to measure template overhead...

* Typically the overhead of starting up a compiler and reading in the standard
  headers dwarfs most template instantiations, so measuring total allocations is
  noisy
   * For clang, use -Xclang -print-stats to get fine detail; it even works in
     Compiler Explorer.
   * Dump all types: compile with "-g", then "dwarfdump -debug-types
     -debug-pubtypes -debug-gnu-pubtypes foo.o"

---
### Compile-time basics: functions

```cpp
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

???
Inefficient at runtime, but compiles quickly.

---
### Compile-time basics: functions

```cpp
*constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  int fib = Fibonacci(15);
  return fib;
}
```

---
### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  `constexpr` int fib = Fibonacci(15);
  return fib;
}
```

---
### Compile-time basics: functions

```cpp
constexpr
int Fibonacci(int n) {
  return (n <= 1) ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
}

int main() {
  // Fails to compile with clang due to "exceeding step limit"
  // Compiles with MSVC but takes 10 seconds; 28 will fail.
  constexpr int fib = Fibonacci(`27`);
  return fib;
}
```

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evaluation for
? : doesn't block template instantiation.

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = n <= 1 ? n : Fibonacci<n - 1>::value +
                                            Fibonacci<n - 2>::value;
};

*template<> struct Fibonacci<0> { static constexpr int value = 0; };
*template<> struct Fibonacci<1> { static constexpr int value = 1; };

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evluation for
? : doesn't block template instantiation.

---
### Compile-time basics: templates

```cpp
template<int n>
struct Fibonacci {
  static constexpr int value = [] {
*   if constexpr(n > 1) {
*     return Fibonacci<n - 1>::value + Fibonacci<n - 2>::value;
*   }
    return n;
  }();
};

int main() {
  constexpr int fib = Fibonacci<46>::value;
  return fib;
}
```

???
Note: Specializations for <0> and <1> were necessary, because lazy evluation for
? : doesn't block template instantiation.

---
#### This one weird trick...

```cpp
template<int N>
constexpr auto Fibonacci() {
  if constexpr (N > 1) {
    return std::integral_constant<int, Fibonacci<N-1>() + Fibonacci<N-2>()>{};
  } else {
    return std::integral_constant<int, N>{};
  }
}

int main() {
  constexpr int fib = Fibonacci<46>();
  return fib;
}
```

???
By putting the calculation into a template instantiation, we force the compiler
to do the calculation at compile-time, even in debug mode.

By making the return value part of the returned type, we make it possible to
avoid calling the routine at all.

The function's signature gets smaller because we're returning auto without the
"->"

---
## The overhead of function signatures

```cpp
std::string add1(std::string a, std::string b);



template<typename T>
T add2(T a, T b);



template<typename T>
auto add3(T a, T b) { return a + b; }
```

---
## The overhead of function signatures

```cpp
std::string add1(std::string a, std::string b);

*add1(std::string, std::string)

template<typename T>
T add2(T a, T b);

*std::string add2<std::string>(std::string, std::string)

template<typename T>
auto add3(T a, T b) { return a + b; }

*auto add3<std::string>(std::string, std::string)
```

---
## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

```

---
## The overhead of function signatures

```cpp
template<typename T>
T process(T a);

*std::string process<std::string>(std::string)

template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

```

---
## The overhead of function signatures

```cpp
template<typename T>
typename std::enable_if_t<!std::is_same<T, void>::value, T>
process(T a);

*std::enable_if<!std::is_same<std::string, void>::value, std::string>::type
*            process<std::string>(std::string)

template<typename T>
auto process(T a) {
  static_assert(!std::is_same<T, void>::value);
  // ...
}

*auto process<std::string>(std::string)
```

---
name: lessons
template: basic-layout

### Lessons so far

1. 'inline' functions aren't necessarily inlined.
   * 'inline' is only guaranteed to affect linkage.
2. 'constexpr' functions are usually not memo-ized, and not necessarily inlined,
   even if they always produce the same result.
   * 'constexpr' only guarantees that the result can be used in situations where
     only a constant is allowed.
3. 'template' instantiations are guaranteed to be memo-ized
   * ... but their functions are not necessarily inlined.
4. 'if constexpr' and 'auto' return types go well with templated functions.
5. Results may vary by compiler.

---
name: custom tuple
template: basic-layout

## Case study: A custom tuple type.

#### Or "How I learned to write an O(N-cubed) template"

```cpp
template<typename... T> class MyTuple;
```

---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<typename... T> class MyTuple;

template<typename T>
class MyTuple<T> {
  T first;
  template<size_t N, typename TN> friend struct Getter;
};

template<typename T, typename... More>
class MyTuple<T, More...> {
  T first;
  MyTuple<More...> more;

  template<size_t N, typename TN> friend struct Getter;
};
```
---
name: custom tuple
template: basic-layout

#### Case study: A custom tuple type.

```cpp
template<size_t N, typename TN> struct Getter {
  template<typename T0, typename... MoreN>
  static TN &get(MyTuple<T0, MoreN...> &tup) {
    return Getter<N - 1, TN>::get(tup.more);
  }
};

template<typename T0> struct Getter<0, T0> {
  template<typename... Ts>
  static T0 &get(MyTuple<Ts...> &tup) { return tup.first; }
};

template<size_t N, typename... Ts>
auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
}
```

???
GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

#### Results: custom tuple type is O(n^3) in codegen

(As measured by clang -fno-inline and objdump --disassemble)

```cpp
using T = MyTuple<bool, char, short, int, double>;

bool tryit4T() {
  T tup;
  get<0>(tup) = true;
  get<1>(tup) = 1;
  get<2>(tup) = 1;
  get<3>(tup) = 1;
  get<4>(tup) = 1.0;
  return get<0>(tup);
}
```

???
GodBolt: https://godbolt.org/z/75Qe2Z

---
name: custom tuple
template: basic-layout

(As seen at https://godbolt.org/z/o2uJWt)

<pre style='font-size:12px'>
tuple_element&lt;0ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;0ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;1ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;1ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;2ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;2ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;3ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;3ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
bool& Getter&lt;0ul, bool>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;1ul, char>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
char& Getter&lt;0ul, char>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;2ul, short>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
short& Getter&lt;1ul, short>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
short& Getter&lt;0ul, short>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;3ul, int>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
int& Getter&lt;2ul, int>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
int& Getter&lt;1ul, int>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
int& Getter&lt;0ul, int>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

Just this one line:
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 double& Getter&lt;4ul, double>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 double& Getter&lt;3ul, double>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 double& Getter&lt;2ul, double>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 double& Getter&lt;1ul, double>::get&lt;int, double>(MyTuple&lt;int, double>&)
 double& Getter&lt;0ul, double>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

1. Change the functions to return auto& to determine the type.

```cpp
template<size_t N, typename... Ts> auto get(MyTuple<Ts...> &tup)
                -> typename std::tuple_element<N, std::tuple<Ts...>>::type& {
  return Getter<N, typename std::tuple_element<N, std::tuple<Ts...>>::type>::get(tup);
```

Becomes this:
```cpp
template<size_t N, typename... Ts> auto& get(MyTuple<Ts...> &tup) {
  return Getter<N>::get(tup);
```
<hr />
<pre style='font-size:20px'>
 tuple_element&lt;4ul, std::tuple&lt;bool, char, short, int, double> >::type&
     get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>
Becomes:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);
</pre>

---
name: custom tuple
template: basic-layout

Still O(N^2) to get a single element however...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls these, in order:
 auto& Getter&lt;4ul>::get&lt;bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&)
 auto& Getter&lt;3ul>::get&lt;char, short, int, double>(MyTuple&lt;char, short, int, double>&)
 auto& Getter&lt;2ul>::get&lt;short, int, double>(MyTuple&lt;short, int, double>&)
 auto& Getter&lt;1ul>::get&lt;int, double>(MyTuple&lt;int, double>&)
 auto& Getter&lt;0ul>::get&lt;double>(MyTuple<double>&)
</pre>

---
name: custom tuple
template: basic-layout

2 . Use function overrides to go directly to an inherited function of your choice

```cpp
template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

---
name: custom tuple
template: basic-layout

One tweak for the base class though:

```cpp
template<typename... T> class MyTuple {
*public:
* void get(...) = delete;
};

template<typename T, typename... More>
class MyTuple<T, More...> : MyTuple<More...> {
  T first;

 public:
  using MyTuple<More...>::get;
  auto& get(std::integral_constant<size_t, sizeof...(More)>) { return first; }
};

template<size_t N, typename... Ts>
auto& get(MyTuple<Ts...> &tup) {
  return tup.get(std::integral_constant<size_t, sizeof...(Ts) - 1 - N>{});
}
```

---

And now...
```cpp
get<4>(tup) = 1.0;
```
Calls:
<pre style='font-size:20px'>
 auto& get&lt;4ul, bool, char, short, int, double>(MyTuple&lt;bool, char, short, int, double>&);

 // Which calls:
 MyTuple&lt;double>::get(std::integral_constant&lt;unsigned long, 0ul>)
</pre>


---
name: custom tuple
template: basic-layout

Still O(N^2) to instantiate MyTuple however.  MyTuple&lt;A, B, C> inherits from
MyTuple&lt;B, C> which inherits from MyTuple&lt;C>.

Solution: inherit from MyTupleElem&lt;A> and MyTupleElem&lt;B> and MyTupleElem&lt;C> etc.

```cpp
template <typename T> struct MyTupleLeaf {
  T _value;
};

template<typename... T> class MyTuple : MyTupleLeaf<T...> {
public:
 auto& get() {  /* ?? */ }
};
```

???

But how does get() work?  And doesn't this have problems if a T is mentioned
twice?

---
name: custom tuple
template: basic-layout

So let's bind a selector value into the types we derive from...

```cpp
template <std::size_t I, typename T>
struct _tuple_leaf {
  T _value;
};
template<typename... T> class MyTuple : MyTupleLeaf<T...> {
public:
 auto& get() {  /* ?? */ }
};
```

Reference: http://talesofcpp.fusionfenix.com/post-22/true-story-efficient-packing

Older code: https://godbolt.org/z/5RZkb5
Newest code: https://godbolt.org/z/vgGT2I

---
name: conclusion
template: title-layout

## Hopefully this at least serves a basis for understanding C++ templates!

---
name: questions
template: title-layout

# Questions?

???

