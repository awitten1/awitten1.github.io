---
layout: post
title:  "C++ Container Exception Safety"
date:   2023-01-01 09:09:55 -0500
categories: C++
usemathjax: true
---

I was recently re-reading Tom Cargill's famous blog [post](https://ptgmedia.pearsoncmg.com/imprint_downloads/informit/aw/meyerscddemo/demo/MAGAZINE/CA_FRAME.HTM) about exception safety in C++.

## The Problem

The problem that the post lays out is that it is impossible to implement a fully generic stack that has an exception-safe pop function that both removes and returns the last value.

One might ask "What is meant by exception safe?"  One possible answer is "If its member functions throw an exception, the state of the stack should be unchanged."

Cargill's post considers the following stack declaration:

```
  template <class T>
  class Stack
  {
    unsigned nelems;
    int top;
    T* v;

  public:
    unsigned count();
    void push(T);
    T pop();

    Stack();
    ~Stack();
    Stack(const Stack&);
    Stack& operator=(const Stack&);
  };
```

The implementation of pop is:

```
  template <class T>
  T Stack<T>::pop()
  {
    if( top < 0 )
      throw "pop on empty stack";     // This exception is ok.
    return v[top--];                  // Copy constructor could throw an exception!
  }
```

Where `top` is an integer pointing to the last occupied index of the buffer.


When the function returns, it has to call the copy constructor of `T` in order to construct a `T` in the stack frame of the caller.  If the copy constructor throws an exception, that will violate the exception safety guarantee of the stack. That is because the copy constructor of `T` is called *after* we mutate the stack (decrementing `top`).  And so, we have mutated the stack, but `pop` has thrown exception.  That is what we want to avoid.

In modern C++, I would probably write `pop` as:

```
  template <class T>
  T Stack<T>::pop()
  {
    if( top < 0 )
      throw "pop on empty stack";
    return std::move(v[top--]);          // Move constructor could throw an exception!
  }
```

This poses the same problem, because the move constructor could also throw an exception.  One option to fix it is to assert that the move constructor will not throw an exception:

```
  template <class T>
  T Stack<T>::pop()
  {
    static_assert(std::is_nothrow_move_constructible_v<T>);
    if( top < 0 )
      throw "pop on empty stack";
    return std::move(v[top--]);
  }
```

Then this `Stack` can only use types with `noexcept` move constructors.  This isn't an option for a `Stack` that wants to be fully generic (the STL containers).  The way the STL handles this is to make functions like `vector<T>::pop_back` be `void`.  If you want to access the back of the `vector` you need to use `vector<T>::back`.

## Would guaranteed NRVO help us?

One comment I want to make is that it appears to me that if Named Return Value Optimization (NRVO) were guaranteed, we would be able to write into the callers stack frame *before* we mutate the stack. (NRVO is a form of [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision).)  For example,


```
  template <class T>
  T Stack<T>::pop()
  {
    if( top < 0 )
      throw "pop on empty stack";
    T copy = v[top];   // Copy constructor called.  Writes into caller stack frame if copy elision occurs!
    --top;
    return copy;       // Copy constructor not called if copy elision occurs!
  }
```

In the above code we make a copy, then we decrement `top`.  If copy elision occurs, then on the `return` line, we don't call the copy constructor!  Instead, it's when we create the copy that we write into the callers stack frame.  The issue is that the C++ standard doesn't guarantee copy elision in this case.

I think that if it were guaranteed, we could have exception safe member functions that both mutate the container and return a value.
