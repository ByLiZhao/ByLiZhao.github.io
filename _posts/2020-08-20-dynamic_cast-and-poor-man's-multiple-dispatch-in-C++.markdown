---
layout: post
mathjax: true
comments: true
title:  "dynamic_cast and poor man's multiple dispatch in C++"
author: John Z. Li
date:   2020-08-15 19:00:00 +0800
categories: c++ programming
tags: dynamic_cast, RTTI, multiple-dispatch
---

C++ only supports single dispatch.
In case one really needs multiple dispatch,
he can emulate it using `dynamic_cast`.
Suppose he needs the following multiple dispatched function:

```cpp
    pets_fight(Pet*, Pet*)
```

The function is required to be dispatched to a
cat-fight-cat, cat-fight-dog, or dog-fight-dog version
according to underlying types of the two parameters.
Also notice that a pet should not be able to fight itself.
This is can be done using dynamic_cast as in the following code:

```cpp
    #include <iostream>
    class Pet {
    public:
      virtual void fight(Pet &other) = 0;
      virtual ~Pet(){};
    };

    class Cat : public Pet {
    public:
      virtual void fight(Pet &other) override;
      virtual ~Cat() override;
    };

    class Dog : public Pet {
    public:
      virtual void fight(Pet &other) override {
        if (dynamic_cast<Dog *>(&other) == this)
          return;
        if (dynamic_cast<Dog *>(&other)) {
          std::cout << "A dog fights another dog." << std::endl;
        } else if (dynamic_cast<Cat *>(&other)) {
          std::cout << "A dog fights a cat" << std::endl;
        } else {
          std::cout << "undefined pets type" << std::endl;
        }
      }

    public:
      virtual ~Dog() override{};
    };

    void Cat::fight(Pet &other) {
      if (dynamic_cast<Cat *>(&other) == this)
        return;
      if (dynamic_cast<Dog *>(&other)) {
        std::cout << "A cat fights a dog." << std::endl;
      } else if (dynamic_cast<Cat *>(&other)) {
        std::cout << "A cat fights another cat. " << std::endl;
      } else {
        std::cout << "undefined pets type." << std::endl;
      }
    }
    Cat::~Cat(){};

    void pets_fight(Pet *pet1_ptr, Pet *pet2_ptr) { pet1_ptr->fight(*pet2_ptr); }

	// test with the following
    int main() {
      Pet *pet1_ptr = new Cat;
      Pet *pet2_ptr = new Dog;
      Pet *pet3_ptr = new Cat;
      Pet *pet4_ptr = new Dog;
      pets_fight(pet1_ptr, pet1_ptr); // fight self is a no-op;
      pets_fight(pet1_ptr, pet2_ptr); // cat fight dog;
      pets_fight(pet3_ptr, pet1_ptr); // cat fight cat;
      pets_fight(pet4_ptr, pet3_ptr); // dog fight dog
      delete pet1_ptr;
      delete pet2_ptr;
      delete pet3_ptr;
      delete pet4_ptr;
      return 0;
    }
```

The code works because when using `dynamic_cast` to cast a pointer to a base class
to a pointer to a derived class, if the pointed object is actually of the type
of the derived class, it will return a valid pointer to an object of that type;
otherwise, it will return a `nullptr`.

*Notice that the same does not apply to references. If a reference of a base class is
cast to a reference of a derived class, an exception will be thrown if the conversion
can not be performed. *
