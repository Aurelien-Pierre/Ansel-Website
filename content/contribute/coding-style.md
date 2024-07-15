---
title: Coding style
date: 2024-07-14
weight: 5
---

## Basic coding logic

Pull requests that don't match the minimum code quality requirements will not be accepted. These requirements aim at ensuring long-term maintainability and stability by enforcing clear, legible code structured with a simple logic.

1. Procedures need to be broken into unit, reusable functions, whenever possible. Exception to this are specialized linear procedures (no branching) doing tasks too specific to be reused anywhere, but in this case use comments to break down the procedures in "chapters" or steps that can be easily spotted and understood.
2. Functions should achieve only one task at a time. For example, GUI code should not be mixed with SQL or pixel-processing code. Getters and setters should be different functions.
3. Functions should have only one entry and one exit point (`return`). The only exceptions accepted are an early return if the memory buffer on which the function is supposed to operate is not initialized or if a thread mutex lock is already captured.
4. Functions should have legible, explicit names and arguments name that advertise their purpose. Programs are meant to be read by humans, if you code for the machine, do it in binary.
5. Functions may only nest up to 2 `if` conditional structures. If more than 2 nested `if` are needed, the structure of your code needs to be reevaluated and probably broken down into more granular functions.
6. `if` should only test uniform cases like the state or the value of ideally one (but maybe more) variable(s) of the same type. If non-uniform cases need to be tested (like `IF user param IS value AND picture buffer IS initialized AND picture IS raw AND picture HAS embedded color profile AND color profile coeff[0] IS NOT NaN`), they should be deferred to a checking function returning a `gboolean` `TRUE` or `FALSE` and named properly so fellow developers understand the purpose of the check without ambiguity on cursory code reading, like `color_matrix_should_apply()`. The branching code will then be `if(color_matrix_should_apply()) pix_out = dot_product(pix_in, matrix);`
7. Comments should mention why you did what you did, like your base assumptions, your reasons and any academic or doc reference you used as a base (DOI and URLs should be there). Your code should tell what you did explicitly. If you find yourself having to explain what your code is doing in comments, usually it's a sign that your code is badly structured, variables and functions are ill-named, etc.
8. Quick workarounds that hide issues instead of tackling them at their root will not be accepted. If you are interested in those, you might consider contributing to upstream darktable instead. The only exceptions will be if the issues are blocking (make the soft crash) and no better solution has been found after some decent amount of time spent researching.
9. Always remember that the best code is the most simple. KISS. To achieve this goal, it's usually better to write code from scratch rather than to try mix-and-matching bits of existing code through heavy copy-pasting.

In an ideal world, any PR would follow [design patterns best practices](https://en.wikipedia.org/wiki/Software_design_pattern).

Some random pieces of wisdom from the internet :

> "Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you can be when you write it, how will you ever debug it?" — Brian W. Kernighan


> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." — Martin Fowler, Refactoring: Improving the Design of Existing Code

> "Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live." — John Woods

> "Whenever I have to think to understand what the code is doing, I ask myself if I can refactor the code to make that understanding more immediately apparent." — Martin Fowler, Refactoring: Improving the Design of Existing Code

> Code is bad. It rots. It requires periodic maintenance. It has bugs that need to be found. New features mean old code has to be adapted. The more code you have, the more places there are for bugs to hide. The longer checkouts or compiles take. The longer it takes a new employee to make sense of your system. If you have to refactor there’s more stuff to move around.
>
> Code is produced by engineers. To make more code requires more engineers. Engineers have n^2 communication costs, and all that code they add to the system, while expanding its capability, also increases a whole basket of costs. You should do whatever possible to increase the productivity of individual programmers in terms of the expressive power of the code they write. Less code to do the same thing (and possibly better). Less programmers to hire. Less organizational communication costs.
> – Rich Skrenta, <http://www.skrenta.com/2007/05/code_is_our_enemy.html>

> Good programmers write good code. Great programmers write no code. Zen programmers delete code. — [John Byrd](https://www.quora.com/profile/John-Byrd-2)

## Specific C coding logic

Ansel as well as darktable are written in C. This language is meant for advanced programmers to write fast bugs in OS and system-level applications. It gives too much freedom to do harmful things and can't be debugged before running the program, or writing your own tests (which can be bugged themselves, or can bias the kind of bugs they let through, and anyway, nobody writes tests). Yet most contributors are not trained for C, many of them are not even professional programmers, so C is a dangerous language for any open source app.

C will let you write in buffers that have not been allocated (resulting in `segfault` error) and will let you free them, but will not free buffers when they are not needed anymore (resulting in memory leaks if you forgot to do it manually), but since buffer alloc/free may be far away (in the program lifetime as in the source code) from where you use them, it's easy to mess that up. C will also let you cast any pointer to any data type, which enables many programmer mistakes and data corruption. The native string handling methods are not safe (*for reasons I never bothered to understand), so we have to use the GLib ones to prevent security exploits.

Basically, C makes you your own and worst enemy, and it's on you to observe safety rules which wisdom will become clear only once you break them. Much like the bugs in a C program. Consider that you write your code to be read by dummies who never programmed in C before.

You also need to keep in mind that the compiler will do most optimizations for you, but will be super conservative about them. The rule of thumb is, if your code is easily understandable by an human (simple logic), it will be properly understood by the compiler, which will take the appropriate optimization measures. The other way around, manual optimizations in the code, that yield cryptic code assumed to be faster on single-threaded systems, usually backfires and yields slower programs after compilation.

1. The control flow of `for` loops should not depend on internal variable's state, that is no conditional `break` or `return` should be used within a `for` loop. Indeed, if the number of loop iterations cannot be planned ahead by the compiler, it will not be able to parallelize efficiently and will be unable to vectorize the code. Breaking loops can happen only for non-computationaly-expensive tasks like finding files.
2. Avoid `while` loops for the same parallelization reasons.
3. C is not an object-oriented language, but you can and should use OO logic when relevant in C by using structures to store data and pointers to methods, then uniform getters and setters to define and access the data.
4. Always access data from buffers using the array-like syntax, from their base pointer, instead of using non-constant pointers on which you perform arithmetic. For example, do:
```C
float *const buffer = malloc(64 * sizeof(float));
for(int i = 0; i < 64; i++)
{
  buffer[i] = ...
}
```
Do not do:
```C
float *buffer = malloc(64 * sizeof(float));
for(int i = 0; i < 64; i++)
{
  *buffer++ = ...
}
```
The latter version is not only less clear to read, but will prevent compiler optimizations and parallelization because the value of the pointer depends on the loop iteration and would need to be shared between threads if any. The former version leads to a memory access logic independent from the loop iteration and can be safely parallelized.

5. The use of inline variable increments (see a [nightmare example here](https://www.youtube.com/watch?v=_7Wok3JoOcE)) is strictly forbidden. These are a mess making for many programming errors.
6. The `case` statements in the `switch` structure should not be additive. Do not do:
```C
int tmp = 0;
switch(var)
{
  case VALUE1:
  case VALUE2:
    tmp += 1;
  case VALUE3:
    do_something(tmp);
    break;
  case VALUE4:
    do_something_else();
    break;
}
```
On cursory reading, it will not be immediately clear that the `VALUE3` case inherits the clauses defined by the previous cases, especially in situations where there are more cases. Do:
```C
int tmp = 0;
switch(var)
{
  case VALUE1:
  case VALUE2:
    do_something(tmp + 1);
    break;
  case VALUE3:
    do_something(tmp);
    break;
  case VALUE4:
    do_something_else();
    break;
}
```
Each case is self-enclosed and the outcome does not depends on the order of declaration of the cases.

7. Sort and store your variables into structures that you pass as function arguments instead of using function with more than 8 arguments. Do not do:

```C
void function(float value, gboolean is_green, gboolean is_big, gboolean has_hair, int width, int height, ...)
{
  ...
}

void main()
{
  if(condition1)
     function(3.f, TRUE, FALSE, TRUE, 80, 90, ...);
  else if(condition2)
     function(3.f, FALSE, TRUE, TRUE, 80, 90, ...);
  else
     function(3.f, FALSE, FALSE, FALSE, 110, 90, ...);
}
```
Do:
```C
typedef struct params_t
{
  gboolean is_green;
  gboolean is_big;
  gboolean has_hair;
  int width;
  int height;
} params_t;

void function(float value, params_t p)
{
  ...
}

void main()
{
  params_t p = { .is_green = (condition1),
                 .is_big = (condition2),
                 .has_hair = (condition1 || condition2),
                 .width =  (condition1 || condition2) ? 80 : 110,}
                 .height = 90 };
  function(3.0f, p);
}
```
The former example is taken from [darktable](https://github.com/darktable-org/darktable/blob/master/src/bauhaus/bauhaus.c#L2210-L2251). The copy-pasting of the function calls is unnecessary and the multiplication of positional arguments makes it impossible to remember which is which. It also doesn't show what arguments are constant over the different branches, which will make refactoring difficult. The latter example is not more concise, however the structure not only makes the function easier to call, but the structure declaration allows to explicitly set each argument, with inline checks if needed. The dependence of the input arguments upon the external conditions is also made immediately clear, and the boolean arguments are directly set from the conditions, which will make the program easier to extend in the future and less prone to programming error due to misunderstandings in the variables dependence.

## Guidelines

1. **Do things you master** : yes, it's nice to learn new things, but Ansel is not a sandbox, it's a production software, and it's not the right place to get your training.
2. **KISS and be lazy** : Ansel doesn't have 50 devs full-time on deck, being minimalistic both in features and in volume of code is reasonable and sane for current management, but also for future maintenance. *(KISS: keep it stupid simple)*.
3. **Do like the rest of the world** : sure, if everybody is jumping out of the window, you have a right to not follow them, but most issues about software UI/UX have already been solved somewhere and in most cases, it makes sense to simply reuse those solutions, because most users will be familiar with them already.
5. **Programming is not the goal** : programming is a mean to an end, the end is to be able to process large volume of pictures in a short amount of time while reaching the desired look on each picture. Programming tasks are to be considered overhead and should be kept minimal, and the volume of code is a liability for any project.