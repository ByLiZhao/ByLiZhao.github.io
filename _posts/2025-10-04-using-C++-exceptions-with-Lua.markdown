---
layout: post
mathjax: true
comments: true
title:  "Using C++ exceptions with Lua"
author: John Z. Li
date:   2025-10-04 20:00:00 +0800
categories: programming
tags: c++, lua, exception
---
When calling C/C++ code from Lua, consider the following cases regarding control flow:
1. Lua calls C/C++ code, and there is an error in C/C++ code.
2. Lua calls C/C++ code, and C/C== code calls Lua code, and there is an error inside the called Lua code.
3. A Lua coroutine calls C/C++ code, and inside C/C++ code the coroutine yields.
4. A Lua coroutine calls C/C++ code, and C/C++ code calls another Lua coroutine, and the innermost coroutine yields.

The default Lua build uses C's `longjmp` to implement non-local control flow.
As a result, Lua's runtime unwinds the call stack of native code without performing any cleanup.
For the above mentioned cases:
1. The called code can perform cleanup before calling `lua_error` or `LuaL_error`, since it is in control when to raise an error.
2. When calling Lua code from C/C++ code, use `lua_pcall` instead of `lua_call`. If there is any error, perform cleanup and raise
   the error again. If you want the caller to see the actual call stack, you need to explicitly generate it (for example, using `luaL_traceback`.
3. The called code can perform cleanup before calling `lua_yield`. Alternatively, `lua_yieldk` can be used to return an extra "continuation function"
   plus an extra integer.  When the Lua coroutine is resumed, that "continuation function" will be called with the integer as its parameter. The
  "continuation function" should be designed in such a way that it can restore state according to the integer value.
4. To handle this case, we need to do everything as in Case 3, as well as checking the return value of `lua_resume`.

We can see that, as native code and Lua code interaction increases, the amount of manual work needed to carefully handle the negative effects
of `longjump` increases unproportionally. The root cause is that when stack is unwound using `longjmp`, there is no way to insert cleanup / restoration
logic into the process. C++'s exception provides a better solution to handle this kind of problems. When Lua is built with a C++ compiler, for Lua5.2 dand
forward, Lua will switch to use C++'s exception mechanism to implement non-local control flow.
(To find where this is defined, search `LUAI_THROW` in Lua source code.)
With this, when an error is raised, a C++ exception is thrown, and the caller captures the exception using `try/catch`.
But note that the exception is a Lua implementation detail, you should never throw an exception of that type from you C++ code directly.
Now things are simplified a lot:

For Case 1: Lua calls C++ code but an error happens in C++ code, just raise an error using `lua_error` or `LuaL_error`.
Destructors will be called automatically during stack unwinding. This means, allocated memory is deallocated, opened files
are closed. If the C++ code is exception safe, the program is in a valid state.

For Case 2: Lua --> C++ code --> Lua code (error happens here), No special attention needs to be paid. The Lua caller code
gets the error with correct stack trace,

Case 3 and Case 4 are more tricky, as compiling Lua with a C++ compiler does not change how Lua coroutines are implemented.
When a Lua coroutine yields, a `longjmp` is always issued. So, if `lua_yield` is called inside C++ code called by Lua coroutine,
destructors won't be called. Though a simple trick can help here: wrap your C++ code into another function, like below
```cpp
extern "C" {
// return 0: normal return
// return positive value: yield
// return negative value: error
int perform_actual_work(lua_State* L) {
try{
// code is wrapped in try block
// do actual work
// Push value into Lua stack
}
catch(std::exception& e) // assuming your C++ code only throw std::exception
						 // As Lua runtime throw int,
						 // if C++ code calls Lua code,
						 // It won't be caught here
	const char * msg = e.what()
    lus_pushstring(L, msg);
	return -1;
}
}

// the function called by
int cpp_fun(lua_State* L) {
	int ret = perform_actual_work(L);
	if(ret > 0){
		return lua_yield(L, 1);  // yield one value
		// will longjmp from here
	}
	else if(ret == 0){
		return 1; // returns one value
	}
	else{
		lua_error(L); // throws C++ exception here
		return 0; // this line is never reached
	}
}
}

```
By the time `lua_yield` is called, the wrapped function `perform_actual_work` has already returned.
If there is any cleanup logic needs to be applied, it should be done inside the function.

We are almost there, but not quite. The next time the Lua coroutine is resumed, the `cpp_fun` won't be
called again. This is where `lua_yieldk` can help us, we modify the signature of `perform_actual_work` into
below
```cpp
int perform_actual_work(lua_State* L, int status, lua_KContext ctx);
```
And the resulting code is something like below:
```cpp
extern "C" {
// return 0: normal return
// return positive value: yield
// return negative value: error
int perform_actual_work(lua_State* L, int status, lua_KContext ctx){
try{
// code is wrapped in try block
// do actual work
// Push value into Lua stack
	my_state * state = nullptr;
	if(status != LUA_OK){
		lus_pushstring(L, "coroutine resumed with error");
		return 0;
	}
	switch((int)ctx){
		// hand rolled state machine
		case 0:
		{
			state = new State{};
			// perform task,
			// push result to Lua stack
			return 1;
		}
		case 1:
		{
			// continue work
			// push result to Lua stack
			return 2;
		}
		// add more cases if you need
		case 2:
		default:
		{
			// finish work
			// push value to Lua stack
			delete state;
			return 0;
		}
	}
}
catch(std::exception& e){ // catch only C++ exceptions
	// clean up since we have error
	delete state;
	const char* msg = e.what();
    lus_pushstring(L, msg);
	return -1;
}
}

// the function called by
int cpp_fun(lua_State* L) {
	int ret = perform_actual_work(L, LUA_OK, 0);
	if(ret > 0){
		return lua_yieldk(L, 1, (lua_KContext)ret, perform_actual_work);  // yield one value
		// will longjmp from here
	}
	else if(ret == 0){
		return 1; // returns one value
	}
	else{
		lua_error(L); // throws C++ exception here
		return 0; // this line is never reached
	}
}
}

```
The code inside `switch` case in the function `perform_actual_work` is a state machine.

Case 4 is not much different than Case 3, but we need to call the Lua coroutine inside
`perform_actual_work`. After checking checking the return code of `lua_resume`, return `0`
on `LUA_OK`, return a positive number on `LUA_YIELD`, and error out otherwise. When errors out,
an error message resulted by calling `lua_resume` is already on the stack. C++ code only needs to forward it.
