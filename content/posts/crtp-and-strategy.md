+++
title = 'Transforming the Strategy Pattern into CRTP in C++'
date = 2024-11-17T03:29:10-05:00
draft = true
+++

In this blog post, I’ll guide you through a journey to understand how the **Curiously Recurring Template Pattern (CRTP)** is connected to the **Strategy Pattern**. We’ll start with the classic Strategy Pattern, modify it step by step, and ultimately uncover how CRTP emerges as its compile-time variant. Each step is only a small modification to the previous version, so you can follow along easily.

> Note: This blog post is not about CRTP itself. If you are not familiar with CRTP and its benefits, I recommend reading [my previous post]() first.

## The Plan of Attack

This transformation can be broken into four stages:

1. **Classic Strategy Pattern**: Dynamic polymorphism using virtual functions.
2. **Static Dispatch with Templates**: Replacing runtime overhead with compile-time polymorphism.
3. **Fusing Strategy and Context**: Simplifying relationships for better optimization.
4. **CRTP in Action**: Achieving zero-overhead polymorphism through templates.

Let’s explore each stage in detail.

## Stage 1: Classic Strategy Pattern

Here we start, from the classic Strategy Pattern. I believe most of you are already familiar with it, but let's have a short recap anyway. The idea is that we have an operation that we want to perform, but the exact implementation of that operation can vary. To achieve this in a pluggable way, we can define different strategies that implement the operation. Then, we can use a context object that is configured with a particular strategy, and the context object can delegate the operation to the strategy.

For instance, consider a system with multiple sorting strategies:

```cpp
struct Sorter {
    virtual void sort(std::vector<int>& data) = 0;
    virtual ~Sorter() = default;
};

struct BubbleSort : public Sorter {
    void sort(std::vector<int>& data) override {
        // Bubble sort implementation
    }
};

struct QuickSort : public Sorter {
    void sort(std::vector<int>& data) override {
        // Quick sort implementation
    }
};

class Context {
    Sorter* sorter;
public:
    Context(Sorter* strategy) : sorter(strategy) {}
    void executeStrategy(std::vector<int>& data) {
        sorter->sort(data);
    }
};
```

This design allows strategies to be dynamically assigned or swapped at runtime. However, runtime polymorphism comes with costs, such as virtual function overhead and potential cache inefficiencies.

---

## Stage 2: Static Dispatch with Templates

To eliminate the cost of virtual functions, we can replace dynamic polymorphism with static polymorphism using templates. This shifts the decision of which strategy to use from runtime to compile time. To allow the context to know which strategy to use, now we need to pass the strategy as a template argument to the context. Here's how it looks:

```cpp
template<typename Sorter>
class Context {
    Sorter sorter;
public:
    Context(Sorter* strategy) : sorter(strategy) {}
    void executeStrategy(std::vector<int>& data) {
        sorter.sort(data);
    }
};

struct BubbleSort {
    void sort(std::vector<int>& data) {
        // Bubble sort implementation
    }
};

struct QuickSort {
    void sort(std::vector<int>& data) {
        // Quick sort implementation
    }
};
```

With templates, `Context` works directly with a `Sorter` type specified at compile time, eliminating the need for virtual functions. This approach achieves better performance, as the compiler can resolve and potentially inline method calls. However, please note that from this point on, strategies must now be chosen at compile time, which trades runtime flexibility for performance.

---

## Stage 3: Fusing Strategy and Context

In many cases, a group of strategies are always used within a specific `Context`. In this case, it would provide the user with more learnable interface, if we can integrate the two together. A potential way is to inherit the strategy from the context template, so they becomes a concrete version of the context. You may think this as follows: "The Strategy is now filled into the Context, so the resulting solid Context *is-a* specific Context to that is specified to run that strategy".

```cpp
template<typename Derived>
class Context {
    Derived* derived;
public:
    Context(Derived* strategy) : derived(strategy) {}
    void executeStrategy(std::vector<int>& data) {
        derived->sort(data);
    }
};

class BubbleSort : public Context<BubbleSort> {
public:
    BubbleSort() : Context(this) {}
    void sort(std::vector<int>& data) {
        // Simple bubble sort implementation
    }
};

class QuickSort : public Context<QuickSort> {
public:
    QuickSort() : Context(this) {}
    void sort(std::vector<int>& data) {
        // Quick sort implementation
    }
};
```

In this design, each strategy class (`BubbleSort`, `QuickSort`) inherits from `Context`, passing itself as the template argument. This tighter coupling eliminates redundancy,  and sets the stage for fully leveraging CRTP.

---

## Stage 4: CRTP in Action

In the previous code, you may notice some redundancy. Specifically, we're passing the derived class instance (`this`) to the Context constructor. However, the Context class already has access to the derived class instance via the `this` pointer. To simplify our code further, we can leverage the `this` pointer directly in the base class to call the appropriate `sort()` function. This is where the CRTP comes in handy. It allows us to remove this duplication and make the code cleaner while retaining compile-time polymorphism. Let's see how we can modify the code to make use of CRTP more effectively:

```cpp
template<typename Derived>
class Context {
public:
    void executeStrategy(std::vector<int>& data) {
        static_cast<Derived*>(this)->sort(data);
    }
};

class BubbleSort : public Context<BubbleSort> {
public:
    void sort(std::vector<int>& data) {
        // Bubble sort implementation
    }
};

class QuickSort : public Context<QuickSort> {
public:
    void sort(std::vector<int>& data) {
        // Quick sort implementation
    }
};
```

Here, the `Context` class uses `static_cast` to call the appropriate `sort()` function from the derived class. By resolving method calls at compile time, CRTP eliminates virtual function overhead and enhances type safety.

## A bit of Conclusion

This transformation demonstrates how CRTP achieves polymorphic behavior with no runtime cost while retaining the clarity and reusability of the Strategy Pattern. Through this journey, we’ve seen how CRTP serves as a compile-time counterpart to the Strategy Pattern, offering both efficiency and elegance in C++ design.

To sum it up, CRTP is like a version of the strategy pattern where the choice of strategy happens when we compile the code instead of when we run it. This makes the program safer and faster, while still letting us switch between different algorithms. It also shows just how powerful C++ templates can be for making flexible and efficient code designs.