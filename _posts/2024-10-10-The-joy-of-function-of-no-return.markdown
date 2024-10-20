---
layout: post
mathjax: true
comments: true
title:  "The joy of functions of no return"
author: John Z. Li
date:   2024-10-10 20:00:00 +0800
categories: c++
tags: c++, inlining, latency
---
Assuming you are working on a toy trading system, it consits the following modules:
1. A client order validation module (let us call it RequestValidator);
2. A risk check module (let us call it RiskChecker);
3. A order book module to do book-keeping (let us call it OrderBook);
4. An Exchange session management module which translates an order into Exchange format and send it to Exchange (let us call it ExhcnageClient).
For each order, it folows through the above modules sequentially until either it being sent to Exchange successively, or it being rejected because an error happens while processing it. To put it into psedo code, it will be like below:
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
Or, if we want to flatten the nested `if-else` cases, we can transform the code into its equivalent form as below (assuming the code is wrapped in a function that returns `void`):
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
Now, instead of nested `if-else` cases, we have a cascading of `if` checks, each handling an error that leads to early return. In realworld systems, this kind of conditional progression is commonplace and the depth (the number of `if`'s along processing a piece of data) can be large. For low-latency software, this kind of code poses some limitation on performance optimization regarding branch prediction and inlining. (No, `[[liekly]]` and `[[unlikely]]` usually won't help with branch prediction here as they only affect the layout of generated machine code. Modern CPU's usually use dynamic branch predictor that ignores static hints at source code level.) It is also hard for the compiler to statically determine which function call should be inlined for better performance. Profiling guided optimization will help here, but that is another topic. 

In this post, the author want to discuss a simple trick that makes our intention obvious at the call graph level. 
That will also allow us for fine-grained tuning afterwards. So, instead of mixing happy path code and error handling code together, we split them into separate functions. Instead of doing 
```
 RequestValidator <----> RiskChecker <----> OrderBook <----> ExchangeClient
 ```
 We do the below as an alternative:
 ```
RequestValidator ----> RiskChecker ----> OrderBook ----> ExchangeClient
     |                    |                |                  |
  (error)        <----  (error)    <---- (error)   <----   (error)
```
where the arrow denotes the direction of data flow, double arraw `<---->` denotes a function call that returns,
and a single arrow `---->` or `<----` denotes a function call that does not return. In pseduo code, it looks something like below
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
We can use compiler intrinsics to mark these error handling fuctions as not being inlined (for GCC, it is `__attribute__ ((noinline))`), for we do not want the error handling code to increase the latency of happy path
latency. Code snippet like `if(error) do_function_call()` followed by some unconditional code, is a clear indication that the `if` branch should not be favored by the CPU branch predictor. The trailing non-return function call
will be translated by the compiler to a simple `jump` machine code if the function is not inlined, or some unconditional code without as if the called function is expanded in the calling function. For most modern C++ compilers, trailing non-return function calls can be heavily optimized (called Tail Call Optimization) by reusing
the call stack of the calling function. For fine-grained optimization, we can force inling these tail calls using compiler intrinsics  (for GCC it is `__attribute__((always_inline))`). Force inlining does not necessarilly improve
latency, for the best arrangement, we need some experiments. 

From a data flow perspective, if data flows from the `process()` function in a upstream module A to its next module B,
it will never flow back to the same function of A. And any code path that involves error handling is guaranteed to have
a deeper call graph than that of the corresponding happy path code. For example, if an order is eventally successfuly sent to Exchange, it will consist of 4 function calls, but an error happenning inside `ExchangeClient` will consit of
8 function calls. A deeper call graph is a clear indication that the corresponding branch is less likely to be taken.
Besides performance benefit, this arrangement of code also has the benefit that a module will not forget to notify its
upstream module that an error has happened.