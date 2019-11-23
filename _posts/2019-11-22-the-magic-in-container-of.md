---
layout:	braindump
title: Magic in the container_of() linux kernel macro
description: container_of()'s magic in linux kernel
excerpt: container_of()'s magic in linux kernel
category: linux
image: /assets/img/containerof.png
---

### TL;DR

- this macro is defined in `<linux/kernel.h>` and returns a pointer to the structure containing a specific member
- it replies to the question of *what is the structure containing this member?*
- the implementation this braindump targets is the one available in [v5.3.12](https://elixir.bootlin.com/linux/v5.3.12/source/include/linux/kernel.h#L964)
- read the related kernel documentation available on [kernel.org](https://www.kernel.org/doc/Documentation/driver-model/design-patterns.txt)
- spoiler alert: no magic involved

If you fancy writing some kernel drivers code or just look around the existing one, you will often notice the usage of the `container_of` macro. But what does it do? And how? The answer to the first question is pretty accessible. Simply put, it takes a structure member and returns a pointer to the structure containing that member. In order to do that it actually takes three arguments: a pointer to the member, the type of the structure (the container) and the name of the member within the structure.

```
  +the type of the structure (second argument)
  |  +returned pointer (return)
  |  |
  |  +------>-----------------------------+
  |          |                            |
  +------------------->struct foo         |
             |                            |
             |    +------------------+    |
             |    |   int memberA    |    |
             |    +------------------+    |
             |    +------------------+    |
  pointer    |    |   int memberB    |    |
  to the     |    +------------------+    |
  member   +----->-------------------+    |
  (first     |    |struct bar memberC|    |
  argument)  |    +----------------^-+    |
             |                     |      |
             +----------------------------+
                                   +
   name of the member within structure
                  (third argument)
```

That is *what* this macro does but the answer to *how* starts with [`cscope -Rk`](http://cscope.sourceforge.net/) searching for `container_of`. The actual implementation is available as part of `<linux/kernel.h>`:

```
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&	\
			 !__same_type(*(ptr), void),			\
			 "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })
```

Let's take the piece of code above and dissect it as much as we can.

## Statement And Declarations in Expressions

The first thing that catches the eye is the fact that even if the macro should return a pointer, it actually has three statements surrounded by `{}`. This is a [GNU C extension](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html)[^1] and has the ability to provide a set of statements, the last one's value serving as the value of the entire construct.

Let's take a simple example:

```
#include <stdio.h>

int main(int argc, char *argv[]) {
	int foo = ({
		int bar = 1;
		bar++;
		if (bar > 1)
			bar++;
		bar;
	});
	printf("%d\n", foo);
}
```

`foo` is defined using a compound statement enclosed in parentheses. The value of this compound statement is the value of the last statement within. That means that the value of `foo` ends up being the value of `bar`. The latter is initialized to `1` and afterward incremented twice. This means that printing `foo` later prints the value `3`. If we reflect back to our `container_of()` macro, we have a pretty similar situation. The macro is using a compound statement as well, containing three statements where the last one's value ends up the return of the macro:

```
void *__mptr = (void *)(ptr);
BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&
		 !__same_type(*(ptr), void),
		 "pointer type mismatch in container_of()");
((type *)(__mptr - offsetof(type, member)));
```

Nothing to comment about the first statement. It defines a pointer to manipulate later. The next two need some more digging.

## Zero Pointer Dereference

The second statement above checks two things:

* if the pointer to the member (first macro argument) is the same as the one of the member name (third macro argument)[^2]
* if the pointer to the member (first macro argument) is a void pointer (generic pointer)

If none of these check out, the build will fail with a message. Now, the interesting thing is a bit of pointer magic: dereferencing a zero pointer in `(type *)0)->member`. The only reason of this is to get the type of the member name (third macro argument). The compiler will not complain because the expression itself will never be evaluated - it's only interested in its type.

For example:

```
#include <stdio.h>

int main(int argc, char *argv[]) {
	struct s {
		int m;
	};
	typeof(((struct s*)0)->m) foo = 1;
	printf("%d\n", foo);
}
```

All the compiler needs to figure out is the type of `foo`. So it doesn't really evaluate the value of `0->m` but only gets its type with `typeof` and uses it for defining `foo`.[^2] With this out of the way, we got to the bottom of the `BUILD_BUG_ON_MSG` statement:

```
BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&
		 !__same_type(*(ptr), void),
		 "pointer type mismatch in container_of()");
```

## offsetof() ANSI C macro

The last statement is definitely the most important because, as we saw above, its value represents the final value of the macro. The only magic there is the `offsetof()` macro. This returns the offset in bytes of a given member within a structure or a union type. You can find it as part of the [standard library](https://en.wikipedia.org/wiki/C_data_types#stddef.h) but kernel doesn't use that so we should find its own definition:

```
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

We now know what the `0->MEMBER` part does: it gets the type of `MEMBER`. The full statement returns the address of `MEMBER` as part of a structure of type `TYPE` that is loaded in memory at address `0`. This ends up being exactly the offset we are looking for.

## Our full container_of() macro

```
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&	\
			 !__same_type(*(ptr), void),			\
			 "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })
```

We have hopefully understood each bit included in this macro. Let's put them all in a picture. This macro returns a pointer (the last statement in the compound statement) computed by subtracting from the address of the member pointer (first argument), the offset of the member name (third argument) relative to the structure type (second argument).

That's it, no more magic.

## References

* [Kernel documentation](https://www.kernel.org/doc/Documentation/driver-model/design-patterns.txt)
* [Kernel source in v5.3.12](https://elixir.bootlin.com/linux/v5.3.12/source/include/linux/kernel.h#L964)
* [cscope tool](http://cscope.sourceforge.net/)
* [gcc extension: Statement And Declarations in Expressions](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html)
* [Referring to a Type with typeof](https://gcc.gnu.org/onlinedocs/gcc/Typeof.html)
* [standard library stddef](https://en.wikipedia.org/wiki/C_data_types#stddef.h)
* [the Magical container_of() Macro by Radek Pazdera](https://radek.io/2012/11/10/magical-container_of-macro/)

---
[^1]: Be aware that this is a `gcc` extension. `clang` also supports but it is not ANSI/ISO C (C++).
[^2]: The types check is actually done by using a `gcc` built in function [`__builtin_types_compatible_p`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html). It is also implemented in `clang`.
