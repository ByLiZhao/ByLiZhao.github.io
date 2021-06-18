---
layout: post
mathjax: true
comments: true
title:  "C++ does multiple inheritance wrong"
author: John Z. Li
date:   2020-09-01 19:04:18 +0800
categories: c++ programming
tags: multiple-inheritance
---

Multiple inheritance could be useful, but C++ does it wrong.
To illustrate the problem, Let us consider the following example.
Say there are two classes we want to inherit from as below.
```cpp
    class employee {
    public:
      employee(std::string s, int NO) : name_{s}, no_(NO) {}
      virtual void do_something() {
        std::cout << "do some employee thing as an employee" << std::endl;
      }
      std::string get_name() { return name_; }
      int )get_NO() { return no_; }

    private:
      std::string name_;
      int no_; // employee number
    };

    class visitor {
    public:
      visitor(std::string s, int NO) : name_(s), no_(NO) {}
      virtual void do_something() {
        std::cout << "do some visitor thing as a visitor" << std::endl;
      }
      std::string get_name() { return name_; }
      int get_NO() { return no_; }

    private:
      std::string name_;
      int no_; // visitor number
    };
```

One thing to notice is that class `employee` and class `visitor` have
conflicting instance variable names and member functions.
Conflicting names
can be divided into three groups:

1. Semantically duplicate names, like the `name_`
instance variable, and `get_name()` member function.
When class `employee` and class `visitor` are multiple inherited,
the derived class, denoting the set of people who are both an employee and a visitor,
can not have too different names, but only one name,
which is supposed to be returned by the `get_name()` member function.
So it is meaningless to have two (potentially different ) versions of the `get_name()`
function in the derived class.
2. Syntactically conflicting names, like the `no_` instance variable and the `get_NO()`
member function.
Despite having the same name, they actually refer to different things.
The instance variable `no_` in class `employee` means an employee number
and that of class visitor means a visitor number,
with their corresponding `get_NO()` function returning them, respectively.
3. Finally, the virtual function `do_something()` in two classes
is both semantically duplicate and syntactically conflict with each other.

The following should be natural:
1.  With semantically duplicate names, the reasonable thing to
do is to merge them (Common Lisp works this way).
It does not make sense that a person who is both an employee and a visitor has two different names.
Keeping two copies of the same member variable also does not make sense.
2. With syntactically
conflicting names, the reasonable thing to do is provide a way to
disambiguate them from each other in derived classes, while the ambiguous reference to
the original member should become unavalible.
3. With conflicting virtual functions, it should be merged but makes it illegal if
the derived class does not override it as if the virtual function in bases classes
are marked as pure virtual.
With some boilerplate code, the latter can be done in C++, by creating proxy classes, as below.

An imaginray syntax achieving all three goals should be look like the below:

(**Not Really Legal C++: Imaginary Syntax**)
```cpp
    class visitor_empleyee : public employee, public visitor {
    public:
      visitor_empleyee(std::string name, int em_no, int vi_no)
          : employee_proxy(name, em_no), visitor_proxy(_, vi_no) {}
      // initialize the member variable name_ only once.

      virtual void do_some_something() override {
        std::cout << "do some employee thing as both an employee and a vistor"
                  << std::endl;
      }
      // not provide an implementation to override base classes' virtual member
      // function do_something() triggers a compilation error.

      // std::string get_name() is implicitly emerged, will return the value of
      //also implicitly emerged "name_"

      using get_employee_NO = employee::get_NO();
	  // rename the base class employee's member function get_NO to get_employee_NO
      using get_visitor_NO = visitor::get_NO();
	  // rename the base class visitor's member function get_NO to get_visitor_NO
      int get_NO() = delete;
	  // make it illegal to call "get_NO()" from an instance of visitor_employee

    private:
      // std::string name_ is implicilty emerged into only one.

      using emplyee_no_ = emplyee::no_;
      // tell the compiler not to merge the member variable no_ in base class employee,
      // instead, rename it to employ_no_

      using visitor_no = visitor::no_;
      // tell the compiler not to merge the member variable no_ in class visitor,
      // instead, rename it to "visitor_no_"
    };
```

With this imaginary syntax, the language will have the following semantics:
```cpp
employee* ep = new visitor_employee("John", 217, 336);
visitor* vp = new visitor_employee("Peter", 218, 337);

ep.get_name(); //OK: returns "John"
vp.get_name(); //OK: returns "Peter"

ep.get_NO(); // OK: calls employee::get_NO();
vp.get_NO(); // OK: calls visitor::get_NO();

dynamic_cast<visitor_employee*>(ep).get_NO(); // compilation error, ambiguous.
dynamic_cast<visitor_employee*>(vp).get_NO(); // Error, as above

dynamic_cast<visitor_employee*>(ep).get_name(); //OK, returns "John"
dynamic_cast<visitor_employee*>(vp).get_name(); //OK, returns "Peter"

dynamic_cast<visitor_employee*>(ep).get_employee_NO(); //OK: returns 217
dynamic_cast<visitor_employee*>(ep).get_visitor_NO(); //OK: returns 336
dynamic_cast<visitor_employee*>(vp).get_employee_NO(); //OK: returns 218
dynamic_cast<visitor_employee*>(vp).get_visitor_NO(); //OK: returns 337

ep.do_something(); //OK: calls visitor_employee::do_something();
vp.do_something(); //OK: calls visitor_employee::do_something();

// true
assert(sizeof(visitor_employee)  < sizeof(visitor) + sizeof(employee));

```

I hope someday a new programming language will do multiple inheritance correct.
It is a little bit too late for C++.

