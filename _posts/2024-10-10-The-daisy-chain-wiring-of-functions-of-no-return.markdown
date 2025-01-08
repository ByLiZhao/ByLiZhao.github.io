---
layout: post
mathjax: true
comments: true
title:  "Daisy chain wiring of functions of no return"
author: John Z. Li
date:   2024-10-10 20:00:00 +0800
categories: c++
tags: c++, inlining, latency
---
Assuming you are working on a toy trading system, it consists of the following modules:
1. A client order validation module (let us call it RequestValidator);
2. A risk check module (let us call it RiskChecker);
3. A order book module to do book-keeping (let us call it OrderBook);
4. An Exchange session management module which translates an order into Exchange
format and send it to Exchange (let us call it ExhcnageClient).
For each order, it flows through the above modules sequentially, until either
being sent to Exchange successfully, or being rejected because an error happens
while processing it in one of the modules.
To put it into pseudo code, it will be like below:
```cpp
if(RequestValidator.process(order)){
    if(RiskChecker.process(order)){
        if(OrderBook.process(order)){
            if(ExchangeClient.process(order)){
                // order successfully sent to Exchange
            }
            else{
                // handle the error that the order can't be sent to Exchange
            }
        }
        else{
            // handle OrderBook error
        }
    }
    else{
        // handle risk checking error
    }
}
else{
    // handle validation error
}
```
If we want to flatten the nested `if-else` cases, we can transform the code into
its equivalent form as below (assuming the code is wrapped in a function that returns `void`):
```cpp
bool success = false;
success = RequestValidator.process(order);
if(not success){
    // handle validation error
    return;
}
success = RiskChecker.process(order);
if(not success){
    // handle risk checking error
    return
}
success = OrderBook.process(order);
if(not success){
    // handle OrderBook error
    return
}
success = ExchangeClient.process(order);
if(not success){
    // handle the error that order can't be successfully sent to Exchange
    return
}
```
Now, instead of nested `if-else` cases, we have a cascading of `if` checks,
each of which handles an error that leads to early return.
In realworld systems, this kind of conditional progression is commonplace and
the depth (the number of `if`'s of the data flow) can be large.
For low-latency software, this kind of code poses some limitation on performance
optimization regarding branch prediction and inlining.
(Note, `[[liekly]]` and `[[unlikely]]` won't help with branch prediction
as they only affect the layout of generated machine code.
And modern CPU's usually use dynamic branch predictor that ignores static hints at
source code level.)
It is also hard for the compiler to statically determine which function call
should be inlined for better performance.
Profiling Guided Optimization(PGO) will help here, but that is another topic.

In this post, the author want to discuss a simple trick that makes our intention
obvious at the call graph level.
This trick will allow us for fine-grained tuning for latency critical code afterwards.
The idea is, instead of mixing happy path code and error handling code together,
we split them into separate functions. Instead of doing
```
 RequestValidator <----> RiskChecker <----> OrderBook <----> ExchangeClient
 ```
 We do the below as an alternative:
 ```
RequestValidator ----> RiskChecker ----> OrderBook ----> ExchangeClient
     |                    |                |                  |
  (error)        <----  (error)    <---- (error)   <----   (error)
```
where the arrow denotes the direction of data flow,
double arraw `<---->` denotes a function call that returns,
and a single arrow `---->` or `<----` denotes a function call that does not return.
Because the function call graph looks like a daisy chain, we call it daisy chain
wiring of no-return functions.
In pseduo code, it looks something like below
```cpp
class RequestValidator{
    void process(Order& order){
        // do order validation
        if(error)[[unlikely]]{
            handle_error(order, error);
        }
        RiskChecker.process(order); // function call of no return
    }

    void handle_error(Order& order, Error error){
        if(error is not module internal error){
            // this function is called by RiskChecker::handle_error();
        }
        else{
            // handle validation error
        }
    }
};

class RiskChecker{
    vlid process(Order& order){
        // do risk checking
        if(error)[[unlikely]]{
            handle_error(order, error);
        }
        OrderBook.process(order); // function call of no return
    }

    void handle_error(Order& order, Error error){
        if(error is not module internal error){
            // this function is called by OrderBook::handle_error();
        }
        else{
            // handle risk checking error
        }
        // notify RequestValidator an error has happened inside RiskChecker
        RequestValidator.handle_error(order, error);
    }
};

class OrderBook{
    void process(Order& order){
        // do OrderBook book-keeping
        if(error)[[unlikely]]{
            handle_error(order, error);
        }
        ExchangeClient.process(order); // function of no return
    }

    void handle_error(Order& order, Error errror){
        if(error is not module internal error){
            // this function is called by ExchangeCliet::handle_error();
        }
        else{
            // handle OrderBook error
        }
        // notify an error has happened inside OrderBook
        RiskChecker.handle_error(order, error);
    }
};

class ExchangeClient{
    void process(Order& order){
        // try send the order to Exchange
        if(error)[[unlikely]]{
            handle_error(order, error);
        }
    }

    void handle_error(Order& order, Error error){
        // notify OrderBook that an error has happened inside ExchangeClient
        OrderBook.handle_error(order, error);
    }
};
```
We can see that error handling logic is now completely separated from happy path code.
We can use compiler intrinsics to mark these error handling fuctions as
not being inlined (for GCC, it is `__attribute__ ((noinline))`),
for we do not want the error handling code to increase the latency of happy path code.
Code snippet like `if(error)[[unlikely]] do_function_call()` followed by some unconditional code,
is a clear indication that the `if` branch is for slow path.
The trailing non-return function call at the end of a function will be
translated by the compiler to a simple `jump` machine code if the called function is not inlined.
This is already faster than a normal function call with return values.
In the case that the called function is inlined,
the generated code will be as if the called function is expanded in place.
As we explicitly forbids inlining error handling code, cache and registers are less likely to
be invalidated in happy path, leading to lower latency.

Furthermore, for modern C++ compilers, trailing non-return function calls at the end
of the enclosing function body can be
optimized further (called Tail Call Optimization) by reusing
the call stack of the calling function. This will further reduce latency for happy path.
We can even do fine-grained manual optimization with Daisy Chian Wiring
by trying forcing inlining for happy path functions (functions named "process" in the above example).
(For GCC it can be done by `__attribute__((always_inline))`). Keep in mind that forcing
inlining does not necessarilly improve latency.
To find the optimal arrangement, some experiments are needed to determine which function
needs to be inlined or use PGO.

From a data flow perspective, if data flows from the `process()` function in module A to module B,
it will never flow back to the same function of A.
The result is that any code path that involves error handling is guaranteed to have
a deeper call graph than that of the corresponding happy path code up to the point
that triggers the error to be handled.
For example, if an order is eventally successfully sent to Exchange,
it will involve 4 function calls, but error handling inside `ExchangeClient`
will consit of 8 function calls.
A deeper call graph is a clear indication that the corresponding code path is
less likely to be taken. Sadly there is not a compiler option to tell the compiler
that the code path with lowest function call depth should be the most optimized performance-wise.
Though I suspect that is already the heuristic used by the compiler when determing
which piece of code is hot.
