+++
title = 'Traps in "for-loop"s: A Deep Dive into Range-Based Loops and Variable Handling'
date = 2024-07-31T19:53:42-04:00
draft = false
+++

Rust and Go are two modern programming languages that have made significant impacts in their respective domains. Rust shines in system programming, with a focus on memory safety without a garbage collector, while Go excels in simplicity and concurrent programming. Despite both being relatively young, these languages have key differences, even in how they handle range-based loops and loop variables.

In this blog post, we’ll explore how Rust and Go differ in their treatment of loop variables, and a subtle but significant change introduced in Go 1.22. Even seasoned developers can stumble over these nuances, but understanding them can save you hours of debugging.

## Rust’s Predictable Variable Handling in Loops

Rust is known for its stringent ownership and borrowing system, which helps ensure memory safety and prevents data races. These principles extend to its range-based `for` loops. In Rust, each iteration creates a **new variable**, and this behavior is crucial for predictable, safe loop execution.

```rust
fn main() {
    let source = vec![1,2,3];
    let mut results: Vec<&i32> = Vec::new();

    for num in &source {
        results.push(num);
    }

    for num in results {
        print!("{} ", *num);
    }
}
```

### Breakdown
In this snippet:

- The variable num is created anew for each iteration of the loop.
- Rust’s design ensures that each iteration is independent. There’s no risk of previous iterations interfering with current ones because each loop variable is fresh.

Rust’s approach is closely tied to its ownership model. Since Rust emphasizes safety, particularly in concurrent programming, creating new variables for each loop iteration avoids the risk of accidentally sharing mutable state across iterations. This is particularly important in ensuring memory safety without a garbage collector.

## Go’s Approach to Loops: Before and After Go 1.22

Go has historically taken a different approach. Prior to Go 1.22, range-based for loops reused the same variable for each iteration. This design could sometimes lead to unexpected behaviors, especially when working with pointers and closures.

Here’s the same program in Go:

```go
package main

func main() {
    numbers := []int{1, 2, 3}
    pointers := []*int{}

    for _, num := range numbers {
        pointers = append(pointers, &num)
    }

    for _, p := range pointers {
		print(*p)
    }
}
```
You might expect the output to be `1 2 3`. However, before Go 1.22, this code actually prints `3 3 3`. 

### Why I am getting this

Before Go 1.22:
Go’s `range` loop reuses the **same** `num` variable for each iteration.
When you take the address of `num` using `&num`, all the pointers in the `pointers` slice point to the same memory location, which ultimately holds the last value assigned in the loop, which is `3`.

### Go 1.22: Fixing the Loop Variable Pitfall

Recognizing the pitfalls of the shared variable approach, Go 1.22 introduced a change. Go now creates a new variable for each iteration, similar to Rust. So the pointers you store point to different memory locations, each holding the correct value. After this change, the previous example behaves as expected, printing `1 2 3`. This fix aligns Go’s behavior with Rust’s, making range-based loops more predictable and preventing hard-to-detect bugs caused by variable reuse.

## Official Language References

For those who want to dig deeper into the technical details, here are the relevant official documents:
- Rust: [The Rust Reference](https://doc.rust-lang.org/stable/reference/expressions/loop-expr.html#iterator-loops) and [The Rust Book](https://doc.rust-lang.org/book/ch03-05-control-flow.html#looping-through-a-collection-with-for)
- Go: [Go 1.22 Release Notes](https://go.dev/doc/go1.22#language) and [Go Language Specification](https://go.dev/ref/spec#For_statements)

## Wrapping Up

Here’s a side-by-side comparison of the behavior:

| **Aspect**                        | **Rust**                          | **Go (pre-1.22)**                | **Go (post-1.22)**                |
|-----------------------------------|-----------------------------------|----------------------------------|-----------------------------------|
| Loop variable behavior            | New variable for each iteration   | Single variable, reused          | New variable for each iteration   |
| Memory safety                     | Ensured by ownership system       | Potential for bugs with pointers | Eliminates pointer/closure bugs   |
| Handling of closures and pointers | Predictable and safe              | Can lead to unexpected behavior  | Now predictable and safe          |

Understanding how different languages handle loop variables can prevent common but subtle bugs, particularly when working with pointers or closures. Rust’s strict ownership model has always provided a robust and predictable mechanism for handling loop variables. Meanwhile, Go’s change in version 1.22 brings its behavior in line with Rust, making it easier to write safe and bug-free code.

So next time you’re writing a loop, remember to take advantage of these language features to avoid tricky pitfalls. With the latest updates, Go developers can now enjoy the same predictability that Rust has long provided. Happy coding!
