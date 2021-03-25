# libmprompt

_Note: The library is under development and not yet complete. This library should not be used in production code._  
Latest release: v0.1, 2021-03-23.

A 64-bit C/C++ library that implements robust and efficient multi-prompt delimited control. 

This is based on _in-place_ growable light-weight stacklets (called `gstack`s) which use 
virtual memory to enable growing the stacklet (up to 8MiB) but start out using 
just 4KiB of committed memory. This means that this library is only available 
for 64-bit systems (currently Windows, Linux, macOS, and various BSD's) 
as smaller systems do not have enough virtual address space.

There are two libraries provided:

- `libmprompt`: the primitive library that provides a minimal interface for
  multi-prompt control. This has well-defined semantics and is the
  minimal control abstraction that can be typed with simple types.
  `libmpromptx` is the C++ compiled variant that integrates exception handling
  where exceptions are propagated correctly through the stacklets.

- `libmphandler`: a small example library that uses `libmprompt` to implement
  efficient algebraic effect handlers in C (with a similar interface as [libhandler]).

Particular aspects:

- The goal is to be fully compatible with C/C++ semantics and to be able to
  link to this library and use the multi-prompt abstraction _as is_ without special
  considerations for signals, stack addresses, unwinding etc. 
  In particular, this library has _address stability_: using the in-place 
  growable gstacks (through virtual memory), these stacks are never moved, which ensures 
  addresses to the stack are always valid (in their lexical scope). There is
  also no special function prologue/epilogue needed as with [split stacks][split]

- A drawback of this approach is that it requires 64-bit systems in order to have enough
  virtual address space. Moreover, at miminum 4KiB of memory is committed per 
  (active) prompt. On systems without "overcommit" we use internal _gpools_ to 
  still be able to commit stack space on-demand using a special signal handler. 

- We aim to support millions of prompts with fast yielding and resuming. If we run
  the `mp_async_test1M` test (in `test/main.cpp`) we simulate an asynchronous
  service which creates a fresh prompt on each connection, enters it and then suspends it
  (simulating waiting for an async result).
  Later it is resumed again where it calls a function that consumes 32KiB stack space, 
  and finally returns. The test simulates 10 million connections with 10000 suspended 
  prompts at any time:
  ```
  async_test1M set up...
  run 10M connections with 10000 active at a time, each using 32kb stack...
  total stack used: 312500.000mb, count=10000000
  elapsed: 1.158s, user: 1.109s, sys: 0.049s, rss: 42mb, main rss: 39mb
  ```
  This is about 8M connections per second (single threaded, Ubuntu 20, AMD5950X),
  where each connection creates a fresh prompt and context switches 4 times.


Enjoy,
  Daan Leijen and KC S.


Todos:
- Proper backtrace support in debuggers.
- Test on arm64.
- Fix exception propagation with MSVC in certain situations with multi-shot prompts.
- ...

## Building

### Linux and macOS

We use `cmake` to build:

```
> mkdir out/debug    # or out/release
> cd out/debug
> cmake ../..
> make
> ./mptest
```

This will build the libraries `libmpromptx.a` and `libmphandlerx.a`.

Pass the option `cmake ../.. -DMP_USE_C=ON` to build the C versions of the libraries
(but these do not handle- or propagate exceptions).

### Windows

We use Visual Studio 2019 to develop the library -- open the solution 
in `ide/vs2019/libmprompt.sln` to build and test.


## The libmprompt Interface

### C Interface

```C
// Types
typedef struct mp_prompt_s  mp_prompt_t;     // resumable "prompts"
typedef struct mp_resume_s  mp_resume_t;     // single-shot resume

// Function types
typedef void* (mp_start_fun_t)(mp_prompt_t*, void* arg); 
typedef void* (mp_yield_fun_t)(mp_resume_t*, void* arg);  

// Continue with `fun(p,arg)` under a fresh prompt `p`.
void* mp_prompt(mp_start_fun_t* fun, void* arg);

// Yield back up to a parent prompt `p` and run `fun(r,arg)` 
void* mp_yield(mp_prompt_t* p, mp_yield_fun_t* fun, void* arg);

// Resume back to the yield point with a result; can be used at most once.
void* mp_resume(mp_resume_t* resume, void* arg);
void* mp_resume_tail(mp_resume_t* resume, void* arg);
void  mp_resume_drop(mp_resume_t* resume);
```

```C
// Multi-shot resumptions; use with care in combination with linear resources.
typedef struct mp_mresume_s  mp_mresume_t;    // multi-shot resume
typedef void* (mp_myield_fun_t)(mp_mresume_t*, void* arg);

void* mp_myield(mp_prompt_t* p, mp_myield_fun_t* fun, void* arg);
void* mp_mresume(mp_mresume_t* r, void* arg);
void* mp_mresume_tail(mp_mresume_t* r, void* arg);
void  mp_mresume_drop(mp_mresume_t* r);
mp_mresume_t* mp_mresume_dup(mp_mresume_t* r);
```

### Semantics

The semantics of delimited multi-prompt control 
can be described precisely:

Syntax:
```
e ::= v              # value
   |  e1 e2          # application
   |  yield m f      # yield to a prompt identified by marker `m`
   |  prompt v       # start a new prompt (passing its marker to `v`)
   |  @prompt m e    # internal: a prompt frame identified by marker `m`

v ::= \x. e          # function with parameter `x` (lambda expression)
   |  ...
```

Evaluation context:
```   
E ::= []            # hole
   |  E e           # evaluate function first
   |  v E           # and then the argument 
   |  @prompt m E   # we can evaluate under a prompt
```
An evaluation context essentially describes the stack+registers where the hole is the current instruction pointer.

Operational semantics:
```
              e ----> e'
(STEP)    -----------------
          E[e] |----> E[e']    
```

We can now keep evaluating inside an expression context using small step transitions:
```
(APP)      (\x. e) v                ---->  e[x := v]                # beta-rule, application
(NEWP)     prompt v                 ---->  @prompt m (v m)          # install a new prompt with a fresh marker `m`
(PROMPT)   @prompt m v              ---->  v                        # returning a result discards the prompt
(YIELD)    @prompt m E[yield m f]   ---->  f (\x. @prompt m E[x])   # yield to prompt `m`, capturing context `E`
```

Note how in (YIELD) we yield with a function `f` to a prompt `m`. This
continues with executing `f` (popping the prompt) but with the argument
`\x. @prompt m E[x]` which is the resumption function: calling it will
restore the prompt and the original execution
context `E` (!), and resume execution at the original yield location.
These primitives are very expressive but can still be strongly
typed in simply typed lambda calculus and are thus sound and
composable:
```
prompt :: (Marker a -> a) -> a               
yield  :: Marker a -> ((b -> a) -> a) -> b   
```

The action given to `prompt` gets a marker that has 
a type `a`, corresponding to the type of the current context `a` (the _answer_ type).
When yielding to a marker of type `a`, the yielded function has type `(b -> a) -> a`,
and must return results of type `a` (corresponding to the marker context).
Meanwhile, the passed in resumption function `(b -> a)` expects an argument
of type `b` to resume back to the yield point. Such simple types cannot be 
given for example to any of `shift`/`reset`, `call/cc`, fibers, or co-routines, 
which is one aspect why we believe multi-prompt delimited control is preferable.

The growable stacklets are used to make capturing- and resuming
evaluation contexts efficient. Each `@prompt m` frame sits
on the top a stacklet from which it can yield and resume 
in constant time. This can for example be used to create
green thread scheduling, exceptions, iterators, async-await, etc.

For a more in-depth explanation of multi-prompt delimited control,
see "_Evidence Passing Semantics for Effect Handler_", Ningning Xie and Daan Leijen, MSR-TR-2021-5 
[link](https://www.microsoft.com/en-us/research/publication/generalized-evidence-passing-for-effect-handlers/).


### Low Level Implementation 

Each prompt starts a growable stacklet and executes from there.
For example, we can have:
```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |
| prompt <------------> |-------------|
|             |         | @prompt B   | 
.             .         |-------------|
.             .         |             |
.             .         | ...         |         
                        | prompt <------------> |------------|
                        .             .         | @prompt C  |
                        .             .         |------------|
                        .             .         | 1+         |
                                                | yield B f  |<< 
                                                .            .
                                                .            .
```   
where `<<` is the currently executing statement.

The `yield B f` can yield directly to prompt `B` by
just switching stacks. The resumption `r` is also
just captured as a pointer and execution continues
with `f(r)`: (rule (YIELD) with `r` = `\x. @prompt B(... @prompt C. 1+[])[x]`)
```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |         (suspended)
| resume_t* r ~~~~~~~~> |-------------|
| f(r)        |<<       | @prompt B   | 
.             .         |-------------|
.             .         |             |
.             .         | ...         |         
                        | prompt <------------> |------------|
                        .             .         | @prompt C  |
                        .             .         |------------|
                        .             .         | 1+         |
                                                | []         |
                                                .            .
                                                .            .
```   

Later we may want to resume the resumption `r` again with
the result `42`: (`r(42)`)

```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |         (suspended)
| resume_t* r ~~~~~~~~> |-------------|
|             |         | @prompt B   | 
| ...         |         |-------------|
| resume(r,42)|<<       |             |
.             .         | ...         |         
.             .         | prompt <------------> |------------|
.             .         .             .         | @prompt C  |
                        .             .         |------------|
                        .             .         | 1+         |
                                                | []         |
                                                .            .
```   
Note how we grew the stacklet 1 without moving stacklet 2 and 3.
If we have just one stack, an implementation needs to copy 
and restore fragments of the stack (which is what [libhandler] does),
but that leads to trouble in C and C++ where stack addresses can temporarily become invalid.
With the in-place growable stacklets, objects on the stack are never moved
and their addresses stay valid (in their lexical scope).

Again, we can just switch stacks to resume at the original
yield location: 
```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |         
| resume_t* r ~~~~~+--> |-------------|
|             |    |    | @prompt B   | 
| ...         |    |    |-------------|
| resume <---------+    |             |   
|             |         | ...         |         
.             .         | prompt <------------> |------------|
.             .         .             .         | @prompt C  |
                        .             .         |------------|
                        .             .         | 1+         |
                                                | 42         |<<
                                                .            .
                                                .            .
```

Suppose, stacklet 3 now returns normally with a result 43:

```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |         
| resume_t* r ~~~~~+--> |-------------|
|             |    |    | @prompt B   | 
| ...         |    |    |-------------|
| resume <---------+    |             |   
|             |         | ...         |         
.             .         | prompt <------------> |------------|
.             .         .             .         | @prompt C  |
                        .             .         |------------|
                        .             .         | 43         |<<
                                                .            .
                                                .            .
```

Then the stacklets can unwind like a regular stack (this is
also how exceptions are propagated):  (rule (PROMPT))

```
(stacklet 1)            (stacklet 2)            (stacklet 3)

|-------------|
| @prompt A   |
|-------------|
|             |
| ...         |         
| resume_t* r ~~~~~+--> |-------------|
|             |    |    | @prompt B   | 
| ...         |    |    |-------------|
| resume <---------+    |             |   
|             |         | ...         |         
.             .         | 43          |<<       (cached for reuse)
.             .         .             .         |------------|
                        .             .         |            |
                        .             .         |            |                                                
                                                .            .
                                                .            .
```

A nice property of muli-prompts is that there is always
a single strand of execution, together with suspended prompts.
The list of prompts form the logical stack and we can have 
natural propagation of exceptions with proper backtraces.


## The libmphandler Interface

A small library on top of `libmprompt` that implements
algebraic effect handlers. Effect handlers give more structure
than basic multi-prompts and are easier to use.

```C
// handle an effect 
void* mpe_handle(const mpe_handlerdef_t* hdef, void* local, mpe_actionfun_t* body, void* arg);

// perform an operation
void* mpe_perform(mpe_optag_t optag, void* arg);

// resume from an operation clause (in a mp_handler_def_t)
void* mpe_resume(mpe_resume_t* resume, void* local, void* arg);
void* mpe_resume_final(mpe_resume_t* resume, void* local, void* arg);
void* mpe_resume_tail(mpe_resume_t* resume, void* local, void* arg); 
void  mpe_resume_release(mpe_resume_t* resume);
```

Handler definitions:

```C
// An action executing under a handler
typedef void* (mpe_actionfun_t)(void* arg);

// A function than handles an operation receives a resumption
typedef void* (mpe_opfun_t)(mpe_resume_t* r, void* local, void* arg);

// Operation kinds can make resuming more efficient
typedef enum mpe_opkind_e {
  ...
  MPE_OP_TAIL,          
  MPE_OP_GENERAL      
} mpe_opkind_t;

// Operation definition
typedef struct mpe_operation_s {
  mpe_opkind_t opkind;  
  mpe_optag_t  optag;   
  mpe_opfun_t* opfun; 
} mpe_operation_t; 

// Handler definition
typedef struct mpe_handlerdef_s {
  mpe_effect_t      effect;         
  mpe_resultfun_t*  resultfun;     
  ...
  mpe_operation_t   operations[8];
} mpe_handlerdef_t;
```