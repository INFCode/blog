+++
title = 'C++ System Error Code: Components and Best Practices'
date = 2024-10-22T23:05:38-04:00
draft = true
+++

In C++11, a new system for error handling was introduced: error codes. This system is useful to provide a `errno` compatible mechanism for errors. It is very lightweight compared to exceptions. It also allows passing the error value around as a parameter, extendable for your own error codes. Special mechanisms are provided to allow checking against similar errors from multiple sources easily, for example different `FILE_NOT_FOUND` errors from different platforms.

In this post, I will start from the traditional C-style `errno` mechanism, and then introduce the new error code system by how it fixes some of the pain points of `errno`.

I will summarize the components and rationale of the error code system introduced in C++11, and provide some good practices for using it. This blog post referenced [the blog post series](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-1.html) by Christopher Kohlhoff who evolves the design. You might also want to give it a look for more details about its design.

## Extending the `errno`

As some background knowledge, the traditional C-style error handling mechanism is done through the `errno` variable. It is a thread-local variable of type `int` that is set by system calls and some library functions to indicate the type of error that occurred. The value of `errno` is the error code number. You can use `strerror(errno)` to get a string representation of the error, or use `perror(const char *)` to print the error message to the standard error stream.

This system is very simple and lightweight, but as something that is built in back in the C89 days, it has some limitations that make it far from ideal in modern C++ programs.

The first issue is that `errno` is a thread-local variable, therefore accessing it requires accessing thread-local storage, which is not very efficient. It also hides the error from the interface, so the user has no way to figure out if a function may cause an error. It also requires the return value of the function to provide an invalid value to indicate an error, and notify the user to check the `errno` value, which is not very clean, and not always possible.

This is actually easy to fix. We can just replace the global `errno` variable with a local variable in the function, and return the error code to the caller. The caller can then check if the error code is non-zero to determine if an error occurred. Here's what our error code type looks like:

```cpp
typedef int error_code;

result my_function(const args& arg, error_code& ec) {
    if (arg.is_valid()) {
        ec = 0; // Success
        return arg.get_result();
    }
    ec = EINVAL;
    return {};
}
```

Note that we are following the convention of using `0` to represent success, and non-zero values to represent errors, which can simplify the error checking code.

The second issue is extensibility. `errno` is a simple `int` variable, thus it is impossible to distinguish errors from different sources. Defining your own error code is possible, but it is impossible to avoid conflicts with errors from other libraries, or distinguish errors from different sources at the user's side. Also the error message is wrapped inside the `strerror` function, and we cannot add error messages to it. The workaround is to define your own error code and error messages in a separate variable, but that requires the user to remember every library's custom error code variable, which is definitely not a good user experience. To make it worse, since every one runs their own `errno`-like variable, it is not standardized, so the user interface might vary from library to library.

So, let's start creating an improved error handling mechanism that is more extensible. We know a single `int` value is not enough, we need to add some more information to it. A simple way is to add a "namespace" marker to it, for example, by attaching a string or a number to identify the source of the error. This is generally doable, but we still have to avoid name conflicts for namespaces, but come up with a universally unique name for each library is still hard to promise. One can use longer identifiers including more information to lower the chance of conflicts, but first there's still a chance of conflicts, and second the identifiers needs to be compared at runtime, which is very inefficient. 

The error code mechanism figured out a better way to do this. Instead of using a hand written namespace marker, it uses the address of a tag object. Each sources defines its custom tag object, and because C++ guarantees that different objects have different addresses, we can use the address of the tag object as the namespace marker. This way we can ensure the uniqueness of the namespace marker, and the comparison can be done by only comparing the pointer, which is much more efficient. The type of the tag object is called `error_category`. Adding it to our error code type, we get the following:

```cpp
class error_category {};

class error_code {
    int value;
    const error_category* category;
};

class my_error_category : public error_category {};

const error_category& get_my_error_category() {
    static my_error_category instance;
    return instance;
}

result my_function(const args& arg, error_code& ec) {
    if (arg.is_valid()) {
        ec = {0, get_my_error_category()}; // Success
        return arg.get_result();
    }
    ec = {EINVAL, get_my_error_category()}; // Error
    return {};
}
```

This allows any source provider to extend the error code system easily, but we still need an extensible alternative for the `strerror` function. Since we already have a separated `error_category` object for each error family, we can add methods to the `error_category` to map error code to their corresponding error message, so each source can define its own `error_category` type, create a static instance of it, and the user can call it when needed. The `<system_error>`'s mechanism includes two of such functions: `error_category::message()` and `error_category::message()`. It looks like this:

```cpp
class error_category {
public:
    virtual const char* name() const = 0;
    virtual const char* message(int ev) const = 0;
};

class my_error_category : public error_category {
public:
    virtual const char* name() const override { return "my"; }
    virtual const char* message(int ev) const override {
        switch (ev) {
            case 0: return "Success";
            case EINVAL: return "Invalid argument";
        }
        return "Unknown error";
    }
};

const error_category& get_my_error_category() {
    static my_error_category instance;
    return instance;
}

/* Everything else unchanged */
```

and at the user side, we can use the `name()` and `message()` method to get the error message:

```cpp
error_code ec = my_function(arg);
if (ec.value() != 0) {
    std::cout << ec.category().name() << ": " << ec.category().message(ec.value()) << std::endl;
}
```

## More on user experience

Now we have a working error code system. But let's take a step back and think about the user experience. There are three roles involved: the developer who defines a source, the developer who reports the error code, and the end user who checks against the error code.

For the developer who defines a source, they need to define a `error_category` type, and provide a static instance of it.

## Error Comparison is Not That Simple

## Some Best Practices
