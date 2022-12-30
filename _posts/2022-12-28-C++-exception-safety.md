---
layout: post
title:  "C++ Container Exception Safety"
date:   2022-12-28 09:09:55 -0500
categories: C++
usemathjax: true
---

I was recently reading Tom Cargill's famous blog [post](https://ptgmedia.pearsoncmg.com/imprint_downloads/informit/aw/meyerscddemo/demo/MAGAZINE/CA_FRAME.HTM) about exception safety in C++.

The problem that the post lays out is that it is impossible to implement a fully generic stack that has an exception-safe pop function that both mutates the stack and returns a value.

One might ask "What is meant by exception safe?"  If its member functions throw an exception, the state of the stack should be unchanged.

The blog considers the following stack declaration:

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
      throw "pop on empty stack";        // This exception is ok.
    return v[top--];                     // Copy constructor could throw an exception!
  }
```

Where `top` is an integer pointing to the last occupied index of the buffer.


When the function returns, it has to call the copy constructor of `T` in order to construct a T in the stack frame of the caller.  If the copy constructor throws an exception, that will violate the exception safety guarantee that the stack remain unchanged if it throws, because `top` has been decremented.  The issue is that the copy constructor of `T` is called *after* we mutate the stack (decrementing `top`).

It's essential that if we throw an exception, we do not mutate the stack.

In modern C++, I would probably write `pop` as:

```
  template <class T>
  T Stack<T>::pop()
  {
    if( top < 0 )
      throw "pop on empty stack";
    return std::move(v[top--]);                     // Move constructor could throw an exception!
  }
```

This is still a problem, because the move constructor could throw an exception!  One option to fix it is to assert that the move constructor will not throw an exception:

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

One additional comment I want to make is that it doesn't look like any of this is a problem if copy elision were guaranteed in this case.  In particular, if Named Return Value Optimization (NRVO) were guaranteed, we are able to write into the callers stack frame *before* we mutate the stack.  For example,


```
  template <class T>
  T Stack<T>::pop()
  {
    if( top < 0 )
      throw "pop on empty stack";
    T copy = v[top];
    --top;
    return copy;       // Copy constructor not called if copy elision occurs!
  }
```

In the above code we make a copy, then we decrement `top`.  If copy elision occurs, then on the `return` line, we don't call the copy constructor!  Instead, it's when we create the copy that we write into the callers stack frame.  The issue is that the C++ standard doesn't guarantee copy elision in this case.

I think that if it were guaranteed, we could have exception safe member functions that both mutate the container and return a value.
