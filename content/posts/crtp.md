+++
title = 'Optimizing C++ Code with CRTP'
date = 2024-11-17T00:42:19-05:00
draft = true
+++


## Introduction
In C++, the choice between static and dynamic dispatch can have a significant impact on performance, especially in hot paths. Understanding when to use each approach and how to optimize for maximum efficiency is key to writing high-performance code. This post will explore the Curiously Recurring Template Pattern (CRTP) as a way to optimize dynamic dispatch, focusing on practical examples and comparing the resulting assembly.

## Static vs Dynamic Dispatch
Dispatch refers to the mechanism that determines which function implementation is executed when a function call is made. In C++, we have two types of dispatch:

- **Static dispatch** happens at compile time and generally results in more efficient code since the compiler knows exactly which function to call. Function inlining and optimizations are possible with static dispatch.
- **Dynamic dispatch** occurs at runtime and is usually implemented using virtual functions. While dynamic dispatch allows for greater flexibility through polymorphism, it introduces an overhead due to the indirection of the vtable lookup. Dynamic dispatch also makes it harder for the compiler to inline the function for better optimization.

### Dynamic Dispatch Example with a Virtual Function
Let's consider a simple example that uses dynamic dispatch with virtual functions. Imagine we have a blending program, and each blending mode has a `blendSingleColor()` method that operates on a single color channel. We want to blend two images in a hot path (e.g., inside a tight loop). Here's a basic implementation:

```cpp
#include <vector>
#include <memory>

class BlendingMode {
public:
    virtual ~BlendingMode() = default;
    virtual int blendSingleColor(int src, int dest) const = 0;
    void apply(const std::vector<int>& src, std::vector<int>& dest) const {
        for (size_t i = 0; i < src.size(); ++i) {
            dest[i] = blendSingleColor(src[i], dest[i]);
        }
    }
};

class NormalBlending : public BlendingMode {
public:
    int blendSingleColor(int src, int dest) const override {
        return src;
    }
};

class MultiplyBlending : public BlendingMode {
public:
    int blendSingleColor(int src, int dest) const override {
        return (src * dest) / 255;
    }
};

void blendWithMode(const BlendingMode& blendingMode, const std::vector<int>& src, std::vector<int>& dest) {
    blendingMode.apply(src, dest);
}

int main() {
    NormalBlending normalBlending;
    MultiplyBlending multiplyBlending;

    std::vector<int> src(1000, 150);
    std::vector<int> dest(1000, 50);

    blendWithMode(normalBlending, src, dest);
    blendWithMode(multiplyBlending, src, dest);

    return 0;
}
```

In this example, each call to `blendSingleColor()` involves a vtable lookup, which adds overhead. This overhead can be significant if `blendSingleColor()` is called repeatedly in a hot path with many pixels to process.

## What is CRTP?
The Curiously Recurring Template Pattern (CRTP) is a technique in C++ where a class inherits from a template instantiation of itself. This allows the compiler to resolve function calls at compile time, effectively replacing dynamic polymorphism with static polymorphism.

Here is a simple CRTP framework:

```cpp
template <typename Derived>
class Base {
public:
    void interface() const {
        static_cast<const Derived*>(this)->implementation();
    }
};

class DerivedA : public Base<DerivedA> {
public:
    void implementation() const {
        // DerivedA specific implementation
    }
};

class DerivedB : public Base<DerivedB> {
public:
    void implementation() const {
        // DerivedB specific implementation
    }
};
```

In this framework, `Base` is a class template that uses the derived class as a template parameter. This allows `Base` to call methods in `Derived` at compile time, enabling the compiler to inline the functions and optimize them effectively.

With CRTP, the `Base` class can always be statically cast to the derived class (`Derived`). This is because the `Base` class knows the exact derived type it is working with at compile time. The use of `static_cast<const Derived*>(this)` enables the compiler to treat the current instance (`this`) as an instance of the derived class. This means that every call to a derived method can be resolved at compile time, providing concrete type information for optimization, such as inlining the derived implementation directly into the caller, avoiding any overhead associated with dynamic dispatch.

Moreover, this compile-time resolution ensures that all polymorphic behaviors are determined early, allowing the compiler to generate the most efficient code possible. Unlike dynamic polymorphism, which requires runtime type checks and vtable lookups, CRTP eliminates these runtime checks by relying on the known derived type during compilation.

## Applying CRTP to the Blending Example
Now, let's modify our original blending example to use CRTP instead of virtual functions:

```cpp
template <typename Derived>
class BlendingModeCRTP {
public:
    void apply(const std::vector<int>& src, std::vector<int>& dest) const {
        for (size_t i = 0; i < src.size(); ++i) {
            dest[i] = static_cast<const Derived*>(this)->blendSingleColorImpl(src[i], dest[i]);
        }
    }
};

class NormalBlendingCRTP : public BlendingModeCRTP<NormalBlendingCRTP> {
public:
    int blendSingleColorImpl(int src, int dest) const {
        return src;
    }
};

class MultiplyBlendingCRTP : public BlendingModeCRTP<MultiplyBlendingCRTP> {
public:
    int blendSingleColorImpl(int src, int dest) const {
        return (src * dest) / 255;
    }
};

template <typename BlendingMode>
void blendWithMode(const BlendingMode& blendingMode, const std::vector<int>& src, std::vector<int>& dest) {
    blendingMode.apply(src, dest);
}

int main() {
    std::vector<int> src(1000, 150);
    std::vector<int> dest(1000, 50);

    NormalBlendingCRTP normalBlending;
    MultiplyBlendingCRTP multiplyBlending;

    blendWithMode(normalBlending, src, dest);
    blendWithMode(multiplyBlending, src, dest);

    return 0;
}
```

In this version, the `apply` function now uses CRTP to call `blendSingleColorImpl()` on the derived class. The compiler can resolve these calls at compile time, allowing for inlining and eliminating the runtime overhead of dynamic dispatch.

## Comparing the Assembly
To see the performance impact, we can compare the assembly generated by the dynamic dispatch version with that of the CRTP version. Here's [an example on godbolt](https://godbolt.org/#z\:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahUAUgBMAIUtXyKxqiIEh1ZpgDC6egFc2TEAA2cncAGQImbAA5PwAjbFIpAHZyAAd0JWIXJi9ffyC0jKyhcMiYtnjEnhSHbCdskSIWUiJcvwDg2vqhRuaiUui4hOT7Jpa2/M6x/ojBiuHqgEp7dB9SVE4uSwBmCNRfHABqM22PADc6klIT3DMtAEE7%2B6JsNlSjF%2BPTogBPVOZ2NhDgARBIEC6YG5PfYsJRKQ5WRhMQgmACyWGwHgASgAVVTHJI2B6pHyxegEVAgJ6HGmHM7oAiYQ4sVLvH4QDBMJREQ7czAgEAXJxkE4eCJEKEWQK89bkXlEfmCy4i07iyXSnDcxaHTncglE%2B60o2HQSkQ4QTIAL2wAH0eQQvsDDloTlZDg7RTLUAA6K3YCCLV3Hay2AjasyE6nG6OaiUAVisBDMcadJyd3JYzlQNvE3NFup5oNI4OwmAAVDcIEQkAQlIsALQ3MnuEQRYCMXJkACSb3oFvWycTyeBctjg6TKcD2wN0ZpEeBUbnSQXD3nrqe0KMcMOUTIbAkiPcbexeMOIEOJLJFIRSJRwHROBPqlFu9I%2B/oh%2BRx9xz%2B2uH1TyXuSlKLu6wiHM2yKtiYHbeN2vYQOKXpykhsbagWAEPLOhykNgRBrEwXrrlhtJrquy7EY8DwwtuqI%2BPQzhsp%2Bd5PmeF6ksBN5HmiGJPqKdEMQQTG3t%2BeI3Jh9xARSVIkTSSGQZg0HtpicGkD27yIeBSiymBPJoTqQh6hGM6zrh%2BGkIR/aoIcZaHPpAD0hwWHGcaUUaZGPBR04bg8Ly9pmQKir8/ysBwXFfjxOBQg89KMhBt4AOrEEgD7%2BhhzFtqlljSgpmUYnKGF8gKQpXKKap/tlyHyoqJUqmKwjqnZ2BahJRq5ZF2DeiybJWaOzVEFOBoeU8SH7hEAatbSRVKsK1yqg1f5ehAPBaKtco8HGWiDTS9mOYaRikMACR0sqZokBBeEvGaOBsIZRCkAFF4JKa%2B5MBs3qgdNtVzfVEqLbGy2rVocqbdthy7Yc%2B3NEdZrfYc53xEQV1NbdXL3Y9/ykC9xjvT5hq0q%2B74ZSYrFMHuB4iSYbm0gJjH0D8xPAKxbD0XTDOU8AlGgQpSXVqlEBk2%2BFPccAcraagfVatTNI88l/Ms4Jwki2LOloVzsk4XhBHOlzy5cMs9DcHG/ABFwOjkOg3AeLYti8qs6yBRY2x8OQRDaPrywANYgJthjcNI/BsD7wOm%2BbltcPwSggMDbtm/r5BwLAKAYDg%2BDEGQlDUHQSKAtwLuCMIYgSJwMhyMIyhqJocfkPoFiGMYpg29Y9jYI4ziuBA7gTAEddhLM5SVCA0iyOkmTtzk3jtCAdej8UTADAPwzDy3bcNNM3fTyvlxr30C9DIky8ZuMk/5HXR8zGU%2B9D9IyxKPbGzcDsewHIFpzfdFjxO8/PhHKKHC3aQH4H9NywnhIzVKEkpIgQ1mcAgLQfASEOAAP3ARiCaaYmo0BYKzaWdI4H4UQfJW8SlYL0DIJpHk4sULgX0hhDBLpvIwIZEybq9MOR3WqsVU6ZUFq3ClFVL63D5p/T4Rqfq6EOHGVAkaU05o/R2ndI6HW05FGenFr6Ag1oAxBibomcMkYNbYTHAmCcqZthOgUiQlSZDSBWXHMOSW8YhyTlwe5CiGthpeSGtRLc8JCbCwisANiUDwp3ggVI4kHFpKgSIS2NspDyFISobpJqLUMLoAuKQYsv8DH42jGZbW4tcGeIXIwqi9waLwlpkJemjNglROsqgnJBooEyTySkyx8TrGJK0jpVC4iDJo0OBkhI2TAq5OwgUiycj1g2VSQNcGTkXLFPcZ5Up3j7ixSZLLPmaD0ocyyvw9q958qDL1II2aPCRGVWSRc0qwjGr6Qie045qUuqsjYcktWZThoPFGiwca%2BiTI0n8R%2BDmhxBZEw5rg6pStAmHAVmzRm6t2l3LquVf84tAZrUOBtLaQZwZ7UOAdGGJ1Zrw3QBdJGx0bp3Qep8TG2M3qdU%2BgqLhlyHn/X6ti4GhxQYEohlDQ6x04YI0ujS14dKMbPXJsyj6q4NY7JSmgyFAS7wqwlvMwa3NEpyzQYimp7NlZVW%2BRso0UzCIMKGnrA2RsTbuwtlbXRds1gbGDNsCw/BY46EWMsJA2AWA4ESAGP2XAA7kCDr7UO/Bw6R2jq7d2vryDe19obLg2x7XV1jQmuOyxE4IHgBAZO6A3gMASJndhpbGCJGADIOu2crpRwgLEB1sQIjNB%2BHnfgbbWCAIAPKxF0JcLt5AMBsA4MIPtTB6YOpwLEHwTMJD0CjrwfgN0G6SGroQXC9QLgrvNtgdQdQfAvBHeKVuDrySxAeoArwOAHX3QIEHVd5BMmxAyNgUEr12xtkTQIIwwAlAADUCDYAAO59pCiOguohxCSFLjBiuGgHX6B4PXEwaBdGGAILEKOkBljoFSOPFd9Y%2B2eotpksZeGQ1dHHm4ZEG80N90vvMKohQx7ZEY%2Bxuee9WMGFozvY%2BeQAhoYEz0aYvHB6ifXifEToxd79yvjwW%2B98S62q4Mbcg0bHVcEOOoAAHIEesgRpCHGAKgayMhvQWHNGnK47rlNesTV7YOobw2RpDg67NUcY5/vzUWtAJbUhlooFQStwXq0gFrdIetDBG3UBbdXHtHaR3Jf7YO4dL6x0TqIFOmdW7sDzsXfQZdI710wU2Obbd2890OsPce09L7z1pvNlem9Pw72Va9cWZ9Ls30fq/Ru39ub/0sEAyB8DkHmDQfkEXeDshEMqGQ9XWu6HG4hmble6jBGiPZBI2RmNlHGTNXgLfVu28O5d1kwYZjcwpPcfHlx2e49JMLC3t0JgvQhNT1E%2Bdj7X2L53be%2BfLj59XtVBU66zgFh1Oae0%2BHPThnjOmfM5Z6Q1nbOEHszsGHTnc1%2BoDUG6g6n3Oufh9wONvn8fJtc2mjNWmvMU5zT69T5HycR2Zx7V9CRMiuGkEAA%3D%3D%3D). In this test, both examples are compiled using gcc 14.2 with `-O2` optimization. You may notice that the CRTP version, all the function calls are already optimized out, and even uses SIMD instructions for batched operation. However, in the virtual function version, the function calls still cannot be optimized, and the resulting program are doing function calls as-is. You may see that in this case, the virtual functions are slow not only because the cost of vtable, but more importantly losing the chance for further optimization.

## Conclusion
CRTP can be a powerful tool when you need to optimize performance in C++ by replacing dynamic polymorphism with static polymorphism. By eliminating vtable lookups, CRTP enables the compiler to inline functions, resulting in faster code execution, particularly in performance-critical sections.

If you have scenarios where runtime polymorphism isn't strictly necessary, CRTP is a great way to achieve polymorphic behavior without the runtime cost.


