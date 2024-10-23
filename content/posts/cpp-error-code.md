+++
title = 'C++ System Error Code: Components and Best Practices'
date = 2024-10-22T23:05:38-04:00
draft = true
+++

In C++11, a new system for error handling was introduced: error codes. This system is useful to provide a `errno` compatible mechanism for errors. It is very lightweight compared to exceptions. It also allows passing the error value around as a parameter, extendable for your own error codes. Special mechanisms are provided to allow checking against similar errors from multiple sources easily, for example different `FILE_NOT_FOUND` errors from different platforms.

In this post, I will summarize the components and best practices for using error codes in C++. This blog referenced [the blog post series](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-1.html) by Christopher Kohlhoff who evolves the design. You might also want to have a look for for more details about its design.

## What's in the `<system_error>` bundle?

All the components of error code are shipped inside the `<system_error>` header. I will classify them into four groups, considering the expected usage of them. I will introduce them in the following sections.

- `error_category`, `error_code`, `error_condition` and `system_error`: These are the core components that defines the errors.
- `make_error_code` and `make_error_condition`: standard way of creating error code and error conditions.
- `generic_category` and `system_category`: These are the standard error categories defined by the C++ standard.
- `is_error_code_enum`, `is_error_condition_enum`, : These are the helpers trait types that error code providers need to implement, and users don't need to care about.

## The Core Components

The `std::error_code` and `std::error_condition` are the core components that defines the errors. Both of them share a very similar interface, and mainly differ in their semantic meanings. `std::error_code` is the type to be returned from operations representing a concrete error. `std::error_condition` is used to represent errors in a more generic way, for example, it can represent all the `ENOENT` errors from different platforms. It is mostly used for end users to check against.

The two types both use a simple int number to represent the underlying error, following the tradition of `errno` and many other error number representation, which use 0 for success and other values for errors.

To allow errors from different sources being used together without any conflict, an extra identifier is used to distinguish them. This is where the `error_category` comes in. `std::error_category` itself is an abstract class, and you need to implement your own concrete class by inheriting from it if you want to define your own error codes.

`std::system_error` is an exception type, which is used to wrap the error codes and throw them as exceptions. With this type, you can simply throw an exception from an `error_code` object at any boundary where you prefer to expose an exception.
