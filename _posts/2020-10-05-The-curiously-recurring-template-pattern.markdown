---
layout: post
mathjax: true
comments: true
title:  "The curiously recurring template pattern (CRTP)"
author: John Z. Li
date:   2020-10-01 19:00:18 +0800
categories: c++ programming
tags: template, CRTP
---
Curiously Recurring Template Pattern (CRTP) refers to a C++ idiom in which
a class inherit from a class template specialization on the inheriting class itself.
It can be used to enforce an interface or plug some extra logic into every class
that derives from the base class template.
The former use case can partially be achieved via virtual functions
(inheriting from an abstract class) with some performance penalty,
the latter can be hard to be achieved without using CRTP.
These two use cases are often used at the same time.
Whenever there is some commonality of a class template that is supposed to be
shared by all specialization of the class template, we can abstract the shared
characteristics into a base class using CRTP.
Without CRTP, C++ libraries like `Eigen` is either impossible, or will become unmaintainable because code repetition.
CRTP works because a function in C++ is only required to have been defined before it actually gets called.

Use case 1 (to enforce a common interface):

Say we have a class template called `matrix`,
which serves as the base class to be inherited via CRTP.
The matrix class has a member function called `linear_solve()`,
which takes a parameter of vector type, solves the linear system `Ax = b`, where `A`
is a matrix instance, and returns `x`.
For (dense) non-symmetric matrices, (dense) symmetric matrices and
(symmetric or non-symmetric) sparse matrices,
there should be different implementation of
the public method `linear_solve()`.
We can use CRTP to enforce the interface.
If one inherits from the matrix class, he has to implement a member
function named `linear_solve`.
Note that CRTP works only when the result of the `sizeof` operator on the
base class template is not related with actual types of template parameters.
```cpp
    using vector = std::vector<double>;
    template <typename T> class matrix {
    public:
      vector linear_solve(vector b) {
      static_cast<T*>(this)->linear_solve(b);
      }
    private:
      matrix() {}
      friend T;
    };

    class symmetric_matrix : public matrix<symmetric_matrix> {
    public:
      vector linear_solve(vector b) {
      //solve Ax = b with A being dense symmetric
      //solution x written to b
      return b;
      }
    };

    class nonsym_matrix : public matrix<nonsym_matrix> {
    public:
      vector linear_solve(vector b) {
      //solve Ax=b with A being a dense nonsymmetric matrix
      //solution x written to b
      return b;
      }
    };

    class sparse_matrix : public matrix<sparse_matrix> {
    public:
      vector linear_solve(vector b) {
      //solve Ax=b with A being a sparse matrix
      //solution x written to b.
      return b;
      }
    };
```
Here the constructor of the matrix class is made private and it befriends deriving class `T`
to prevent users to accidentally mismatch the class name of the deriving
class and the name of class template parameter. For example, the following triggers a compilation error.
```cpp
    class sparse_matrix: public matrix<symmetric_matrix>
```

Use case 2 (to plug extra logic into derived class)

Say we have a class called `my_class` which has a member function `do_something()`.
We want to create a debug version of the member function `do_something()`
but we don’t want the debug code pollutes readability of our implementation of it.
We can use CRTP to hide the extra logic of debug mode into the base class template,
and plug it into `my_class`, as below,
```cpp
    template <typename T> class debug {
    public:
      void debug_do_something(std::string s) {
      std::cout << "check pre-conditions" << std::endl;
      //logging if pre-conditions not satisfied
      static_cast<T*>(this)->do_something(s);
      std::cout << "check post-conditions" << std::endl;
      //logging if post-condition not satisfied
      }
    private:
      debug() {}
      friend T;
    };

    class my_class : public debug<my_class> {
    public:
      void do_something(std::string s) {
      std::cout << "do real work with " << s.c_str() << std::endl;
      }
    };
```
`boost::operator` relies on this trick to plug a set of overloaded operators into a class.
