<pre class='metadata'>
Title: Calling non-virtual non-static member functions outside lifetime
Shortname: D####
Revision: 0
!Draft Revision: 0
Audience: EWG, LWG
Status: D
Group: WG21
URL:
!Source: <a href="https://github.com/ecatmur/call-nvnsmf-outside-lifetime/blob/main/paper.md">github.com/ecatmur/call-nvnsmf-outside-lifetime/blob/main/paper.md</a>
!Current: <a href="https://htmlpreview.github.io/?https://github.com/ecatmur/call-nvnsmf-outside-lifetime/blob/r0/D####R0.html">github.com/ecatmur/call-nvnsmf-outside-lifetime/blob/r0/D####R0.html</a>
Editor: Ed Catmur, ed@catmur.uk
Markup Shorthands: markdown yes, biblio yes, markup yes
Abstract:
    Calling non-virtual non-static member functions on objects whose lifetime has ended or has not yet begin is undefined behavior.
    We propose to relax this restriction.
Date: 2022-12-25
</pre>
<pre class='biblio'>
{
    "P2280R4": {
        "title": "Using unknown pointers and references in constant expressions",
        "href": "https://wg21.link/P2280R4"
    },
    "cwg1530": {
        "title": "1530. Member access in out-of-lifetime objects",
        "href": "https://wg21.link/cwg1530"
    }
}
</pre>

## 1. Changelog

: v0
:: Initial submission

## 2. Motivation and Scope

What can we do with a `std::array` that is outside its lifetime?
We can't find out its size by calling `size()` on it, since 6.7.3 <a href="https://wg21.link/basic.life#7.2">\[basic.life]/7.2</a>
makes such a member function call undefined, even though `std::array::size()` has no reason to access the object;
so we have to use `decltype` and `std::tuple_size` or some other convoluted solution:

```c++
union { int i; std::array<int, 1> a; } u = {.i = 0};
std::size_t z1 = u.a.size();  // UB
std::size_t z2 = std::tuple_size_v<decltype(u.a)>;  // OK
std::size_t z3 = decltype([] { return std::integral_constant<std::size_t, u.a.size()>(); }())::value;  // OK
```

Similarly, we can't form a glvalue or pointer to one of its elements via `operator[]`, but there's nothing to prevent us calling `std::get`:

```c++
struct S {
    S(int) : p(&a[0]) {}  // UB
    S(long) : p(&std::get<0>(a)) {}  // OK
    int* p;
    std::array<int, 1> a;
};
```

Or, similarly:

```c++
union { int i; std::array<std::monostate, 1> a; } u = {.i = 0};
auto m1 = u.a[0];  // UB
auto m2 = std::get<0>(u.a);  // OK
auto [m3] = u.a;  // OK
```

This restriction feels unnecessary; other than when calling a virtual member function or a member function of a virtual base,
in which cases the vtable must be well-formed, there is very little to distinguish the object parameter of a member function
call from any other parameter, especially now that C++ has explicit object parameters.

Indeed, there is implementation divergence on whether the undefined behavior is noticed during constant evaluation:

```c++
consteval int f() {
    union { int i; std::array<int, 1> a; } u = {.i = 0};
    return u.a.size();  // #1
}
struct S {
    consteval S() : z(a.size()) {}  // #2
    std::size_t z;
    std::array<int, 1> a;
};
constinit auto j = S().z;
```

clang rejects #1 and #2, but with fragility; minuscule changes cause it to accept.
gcc accepts #1 and rejects #2, again with fragility.
MSVC accepts both.

At runtime (outside constant evaluation), the sanitizers do not detect this as a reportable error.

Given that this rule appears to exist merely for the sake of it, and compilers have trouble applying it, we feel justified in calling for its removal.

## 3. Impact On the Standard

This is a pure language extension.

16.4.5.10 <a href="https://wg21.link/res.on.objects">\[res.on.objects]</a> states (somewhat redundantly) that access to library
objects outside their lifetime has undefined behavior unless otherwise specified.
This would continue to hold, as would calls to *virtual* member functions and to member functions of virtual bases;
if LWG wishes, it would be possible to tighten this to disallow calling non-static member functions on *library* objects.
However, we caution against introducing an inconsistency between member and non-member functions, or alternatively
disallowing seemingly innocuous code such as `std::addressof(u.a)` on an object outside its lifetime.

## 4. Technical specification

Amend 6.7.3 \[basic.life] paragraph 7 as follows:

<quote>
... The program has undefined behavior if:

* the glvalue is used to access the object, or
* the glvalue is used to call a <del>non-static</del><ins>virtual</ins> member function of the object
<ins> or a non-static member function of a virtual base class</ins>,
<ins>[Footnote: Evaluation of a non-virtual member function, or initialization of its parameters,
including its explicit object parameter, if any, can also result in undefined behavior. --end footnote]</ins> or
* the glvalue is bound to a reference to a virtual base class (\[dcl.init.ref]), or
* the glvalue is used as the operand of a dynamic_­cast (\[expr.dynamic.cast]) or as the operand of typeid.
</quote>

Update `__cpp_constexpr` to the year and month of adoption.

## 5. Acknowledgements
