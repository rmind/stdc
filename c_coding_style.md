# The coding style for modern C

23 June 2021 (upstream at: https://github.com/rmind/stdc)

## Table of contents

1. [General](#general)
	1. [Indentation](#indentation)
	2. [Line length](#line-length)
	3. [Wrap around](#wrap-around)
	4. [Comments](#comments)
	5. [Spacing and brackets](#spacing-and-brackets)
3. [Declarations and definitions](#declarations-and-definitions)
	1. [Variable names and declarations](#variable-names-and-declarations)
	2. [Type definitions](#type-definitions)
	3. [Initialization](#initialization)
4. [Control structures](#control-structures)
	1. [General](#general-1)
		1. [Avoid unnecessary nesting levels](#avoid-unnecessary-nesting-levels)
	2. [if-statements](#if-statements)
		1. [Avoid redundant else](#avoid-redundant-else)
	3. [switch-statements](#switch-statements)
5. [Functions](#functions)
	1. [Function declarations](#function-declarations)
	2. [Function definitions](#function-definitions)
6. [Program organization](#program-organization)
	1. [Headers](#headers)
7. [Decent coding practices](#decent-coding-practices)
	1. [Write clear, concise, succinct, unambiguous code](#write-clear-concise-succinct-unambiguous-code)
	3. [Use good abstractions](#use-good-abstractions)
		1. [Avoid magic numbers](#avoid-magic-numbers)
		2. [Avoid excessive #ifdef](#avoid-excessive-ifdef)
		3. [Be sensible with goto](#be-sensible-with-goto)
	4. [Error handling](#error-handling)
	5. [String handling](#string-handling)
		1. [Embrace abstractions](#embrace-abstractions)
		2. [Global strings](#global-strings)
	6. [Use reasonable types](#use-reasonable-types)
	7. [Embrace portability](#embrace-portability)
		1. [Byte-order](#byte-order)
		2. [Types](#types)
		3. [Avoid unaligned access](#avoid-unaligned-access)
		4. [Avoid extreme portability](#avoid-extreme-portability)
8. [Reference](#references)

## General

_Controlling complexity is the essence of computer programming._
-- Brian Kernighan

This coding style is a variation of the K&R style, with some elements from
the BSD _Kernel Normal Form (KNF)_, SunOS kernel style and other modifications.

Some general principles: honor tradition, but accept progress; be consistent;
embrace the latest C standards; embrace modern compilers, their static analysis
capabilities and sanitizers.

### Indentation

Use **tabs** rather than spaces.  The tabs shall be **8** characters long.

### Line length

All lines should generally be within 80 characters.  Wrap long lines.
There are some good reasons behind this:
* It forces the developer to write more succinct code;
* Humans are better at processing information in smaller quantity portions;
* It helps users of vi/vim (and potentially other editors) who use
vertical splits.

### Wrap around

Add 4 spaces when wrapping around a long line.  Function declarations may
be wrapped with different spacing such that the parameters would be better
aligned with the function name or its parameter (see below).

Use your discretion on indenting, but be consistent.

### Comments

Multi-line comments shall have the opening and closing characters
in a separate line, with the lines containing the content prefixed by a space
and the `*` characters for alignment, e.g.:
```c
	/*
	 * This is a multi-line comment.
	 */

	/* One line comment. */
```
Use multi-line comments for more elaborative descriptions or before more
significant logical block of code.

Single-line comments may be written in C99 style:
```c
	return (uintptr_t)val;  // return a bitfield
```
Leave two spaces between the statement and the inline comment.

### Spacing and brackets

Use one space after the conditional or loop keyword, no spaces around
their brackets, and one space before the opening curly bracket.

Functions (their declarations or calls), `sizeof` operator or similar
macros shall not have a space after their name/keyword or around the
brackets, e.g.:
```c
unsigned total_len = offsetof(obj_t, items[n]);
unsigned obj_len = sizeof(obj_t);
```

Use brackets to avoid ambiguity and with operators such as `sizeof`,
but otherwise avoid redundant or excessive brackets.

## Declarations and definitions

### Variable names and declarations

- Use descriptive names for global variables and short names for locals.
Find the right balance between descriptive and succinct.

- Use ["snakecase"](https://en.wikipedia.org/wiki/Snake_case).
Do not use "camelcase".

- Do not use Hungarian notation or other unnecessary prefixing or suffixing.

- Use the following spacing for pointers:
```c
const char *name;  // const pointer; '*' with the name and space before it
conf_t * const cfg;  // pointer to a const data; spaces around 'const'
const uint8_t * const charmap;  // const pointer and const data
const void * restrict key;  // const pointer which does not alias
```

### Type definitions

Declarations shall be on the same line, e.g.:
```c
typedef void (*dir_iter_t)(void *, const char *, struct dirent *);
```

_Typedef_ structures rather than pointers.  Note that structures can be kept
opaque if they are not dereferenced outside the translation unit where they
are defined.  Pointers can be _typedefed_ only if there is a very compelling
reason.

New types may be suffixed with `_t`.  Structure name, when used within the
translation unit, may be omitted, e.g.:

```c
typedef struct {
	unsigned	if_index;
	unsigned	addr_len;
	addr_t		next_hop;
} route_info_t;
```

### Initialization

Embrace C99 structure initialization where reasonable, e.g.:
```c
static const crypto_ops_t openssl_ops = {
	.create		= openssl_crypto_create,
	.destroy	= openssl_crypto_destroy,
	.encrypt	= openssl_crypto_encrypt,
	.decrypt	= openssl_crypto_decrypt,
	.hmac		= openssl_crypto_hmac,
};
```

Embrace C99 array initialization, especially for the state machines, e.g.:
```c
static const uint8_t tcp_fsm[TCP_NSTATES][2][TCPFC_COUNT] = {
	[TCPS_CLOSED] = {
		[FLOW_FORW] = {
			/* Handshake (1): initial SYN. */
			[TCPFC_SYN]	= TCPS_SYN_SENT,
		},
	},
	...
}
```

## Control structures

Try to make the control flow easy to follow.  Avoid long convoluted logic
expressions; try to split them where possible (into inline functions,
separate if-statements, etc).

### General

The control structure keyword and the expression in the brackets should be
separated by a single space.  The opening curly bracket shall be in the
same line, also separated by a single space.  Example:

```c
	for (;;) {
		obj = get_first();
		while ((obj = get_next(obj)) != NULL) {
			...
		}
		if (done) {
			break;
		}
	}
```

Do not add inner spaces around the brackets. There should be one space after
the semicolon when `for` has expressions:
```c
for (unsigned i = 0; i < __arraycount(items); i++) {
	...
}
```

#### Avoid unnecessary nesting levels

Avoid:
```c
int
inspect(obj_t *obj)
{
	if (cond) {
		...
		...
		// long code block
		...
		...
		return 0;
	}
	return -1;
}
```
Consider:
```c
int
inspect(obj_t *obj)
{
	if (!cond) {
		return -1;
	}
	...
	...
	...
	return 0;
}
```
However, do not make logic more convoluted.

### `if` statements

Curly brackets and spacing follow the K&R style:
```c
	if (a == b) {
		..
	} else if (a < b) {
		...
	} else {
		...
	}
```

Simple and succinct one-line if-statements may omit curly brackets:
```c
	if (!valid)
		return -1;
```
However, do prefer curly brackets with multi-line or more complex statements.
If one branch uses curly brackets, then all other branches shall use the
curly brackets too.

Wrap long conditions to the if-statement indentation adding extra 4 spaces:
```c
	if (some_long_expression &&
	    another_expression) {
		...
	}
```

#### Avoid redundant `else`

Avoid:
```c
	if (flag & F_FEATURE_X) {
		...
		return 0;
	} else {
		return -1;
	}
```
Consider:
```c
	if (flag & F_FEATURE_X) {
		...
		return 0;
	}
	return -1;
```

#### `switch` statements

Switch statements should have the `case` blocks at the same indentation
level, e.g.:
```c
	switch (expr) {
	case A:
		...
		break;
	case B:
		// fallthrough
	case C:
		...
		break;
	}
```

If the case bock does not break, then it is strongly recommended to add a
comment containing "fallthrough" to indicate it.  Modern compilers can also
be configured to require such comment (see gcc `-Wimplicit-fallthrough`).

## Functions

### Function declarations

In a function declaration, the type _specifiers_ and type _qualifiers_ shall
be in the same line as the function name and its parameters.  They may be
separated by a tab (or multiple) for a better visual separation.  The parameter
names should be omitted.

```c
ssize_t		hex_write(FILE *, const void *, size_t);
char *		hex_write_str(const void *, size_t);
```

Long lines should be wrapped; the continuing parameters may be indented
with tabs up to the function name and then 4 extra spaces:
```c
typedef struct crypto_ops {
	int		(*create)(struct crypto *);
        void		(*destroy)(struct crypto *);
        ssize_t		(*encrypt)(const crypto_t *, const void *,
			    size_t, void *, size_t);
	ssize_t		(*decrypt)(const crypto_t *, const void *,
			    size_t, void *, size_t);
} crypto_ops_t;
```
Alternatively, they may be indented with 4 extra spaces only.  Use your
discretion on indenting, but be consistent.

In the spirit of C99, denote functions without parameters (as opposed to
functions with unknown parameters) using `void`, e.g.:
```c
void	lib_init(void);
```

Do not use old style K&R style C declarations.

### Function definitions

In a function definition, the type _specifiers_ and type _qualifiers_ shall
be in a separate line prior the function name and its parameters.  The opening
and closing curly brackets shall also be in the separate lines (K&R style).
The function definition therefore takes at least four lines:

```c
ssize_t
hex_write(FILE *stream, const void *buf, size_t len)
{
	...
}
```

Do not use old style K&R style C definitions.

## Program organization

### Headers

Order of header inclusion: system level, standard libraries, 3rd party
libraries or dependencies and, lastly, project (local) headers.

Ensure there is a guard for double inclusion, e.g.:
```c
#ifndef _UTILS_H_
#define _UTILS_H_

...

#endif
```

If the header is for public API and may be used C++ application, make sure
to use the `extern "C"` wrappers which can be borrowed from `sys/cdefs.h`:
```c
__BEGIN_DECLS

int	mylib_get_version(void);

__END_DECLS
```

## Decent coding practices

### Write clear, concise, succinct, unambiguous code

_Everyone knows that debugging is twice as hard as writing a program in the
first place. So if you're as clever as you can be when you write it,
how will you ever debug it?_ -- Brian Kernighan

### Use good abstractions

C does not have various features builtin, e.g. more granular visibility
control (public/private), classes, modules, etc.  However, good C programmers
follow the universal principles which such language features help to achieve:
information hiding principle, separation of concerns, code re-usability and
modular design, etc.

#### Avoid magic numbers

Unnamed numerical constants, also called _magic numbers_, should be avoided.

#### Avoid excessive `#ifdef`

See the wonderful essay by
[Henry Spencer from 1992](https://www.usenix.org/legacy/publications/library/proceedings/sa92/spencer.pdf).

Separate the code into low-level and higher-level APIs.  Generalize and
abstract them as needed to avoid CPU-architecture or library-specific ifdefs
in the higher-level layers.

#### Be sensible with `goto`

In addition to
[Go To Statement Considered Harmful](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf)
(1968) by [Edsger Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra), please also see the wonderful essay
[Structured Programming with go to Statements](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.103.6084&rep=rep1&type=pdf)
(1978) by [Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth).

### Error handling

Generally, use the traditional UNIX error handling: return `0` on success
and `-1` on failure; set `errno` on error if needed.  Separate system-level
errors from application-level errors.  Consider the illustrative example:
```c
int http_request(http_request_t *req, http_response_t *resp);
```

System-level errors may include memory allocation failure, hitting the
resource limits, no permission to open a socket, etc.  All these ought to
return a traditional `-1` with the `errno` set.

In this case, application-level errors are basically the HTTP errors.
Receiving HTTP 403 (Forbidden) does not mean it is a system level failure.
The HTTP request still succeeded, therefore the function should return `0`
as it was successful in itself.  The HTTP error should, instead, be reported
in the response structure.

On the other hand, returning HTTP status as a positive number may also be
an option as it is not unprecedented to give meaning for positive values,
e.g. consider `open(2)` or `write(2)` syscalls.

### String handling

#### Embrace abstractions

String handling available in the standard C library provides interfaces
which reflect their historical era of 1970s and 1980s.  While they are
suitable as primitives and building blocks, they are no longer suitable
as a first-class API.

- Implement your own generic string abstractions or use good libraries,
if there is a significant need for string operations in the particular
code base.

- Do not open-code string manipulation in the middle of other unrelated
logic.  For example: do not open-code a loop with `tolower()`; instead,
add `str_tolower()` primitive; do not open-code a loop with `strpbrk()`;
instead, implement a generic `str_split()` primitive.

- Consider using "string buffer" abstraction for slicing, copying, reference
counting or performing other operations on the strings.

- Where suitable, consider using Pascal-like strings (as an abstraction)
for efficiency.

#### Global strings

Global or static strings may defined using the array notation, e.g.:
```c
const char hver[] = "HTTP/1.1";
// You can also get the string length in constant time, without using strlen():
const unsigned hver_len = sizeof(hver) - 1;
```

Side note: pointers have their own memory address, while arrays do not have
a distinct address (i.e. they only have the address of the data they point)
and can lead to some micro optimizations.

### Use reasonable types

Use `unsigned` for general iterators; use `size_t` for general sizes; use
`ssize_t` to return a size which may include an error.  Of course, consider
possible overflows.

Avoid using `uint8_t` or `uint16_t` or other sub-word types for general
iterators and similar cases, unless programming for micro-controllers or
other constrained environments.

C has rather peculiar _type promotion rules_ and unnecessary use of sub-word
types might contribute to a bug once in a while.

### Embrace portability

#### Byte-order

Do not assume x86 or little-endian architecture.  Use endian conversion
functions for operating the on-disk and on-the-wire structures or other
cases where it is appropriate.

#### Types

- Do not assume a particular 32-bit vs 64-bit architecture, e.g. do not
assume the size of `long` or `unsigned long`.  Use `int64_t` or `uint64_t`
for the 8-byte integers.

- Do not assume `char` is signed; for example, on ARM it is unsigned.

- Use C99 macros for constant prefixes or formatting of the fixed-width
types.

Use:
```c
#define	SOME_CONSTANT	(UINT64_C(1) << 48)
printf("val %" PRIu64 "\n", SOME_CONSTANT);
```
Do not use:
```c
#define	SOME_CONSTANT	(1ULL << 48)
printf("val %lld\n", SOME_CONSTANT);
```

#### Avoid unaligned access

Do not assume unaligned access is safe.  It is not safe on ARM, MIPS,
PowerPC and various other architectures.  Moreover, even on x86 unaligned
access is slower.

#### Avoid extreme portability

Unless programming for micro-controllers or exotic CPU architectures,
focus on the common denominator of the modern CPU architectures, avoiding
the very maximum portability which can make the code unnecessarily cumbersome.

Some examples:
- It is fair to assume `sizeof(int) == 4` since it is the case on all modern
mainstream architectures.  PDP-11 era is long gone.
- Using `1U` instead of `UINT32_C(1)` or `(uint32_t)1` is also fine.
- It is fair to assume that `NULL` is matching `(uintptr_t)0` and it is fair
to `memset()` structures with zero.  Non-zero NULL is for retro computing.

Use common sense.

## References

- 1988, Brian W. Kernighan and Dennis M. Ritchie, The C Programming Language, 2nd Edition, Prentice Hall.
- [1993, Bill Shannon, C Style and Coding Standards for SunOS. Sun Microsystems.](https://www.cis.upenn.edu/~lee/06cse480/data/cstyle.ms.pdf)
- 1999, Brian W. Kernighan and Rob Pike, The Practice of Programming, Addison–Wesley.
- [Kernel Normal Form (KNF)](https://www.freebsd.org/cgi/man.cgi?query=style&sektion=9) from FreeBSD Kernel Developer's Manual
- [Linux kernel coding style](https://www.kernel.org/doc/html/v4.10/process/coding-style.html)
