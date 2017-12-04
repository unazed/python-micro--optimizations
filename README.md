# python-micro--optimizations
a handful of optimizations, tricks and cool things i've learned while doing python


# return statements in single else clauses at the end of functions


```py
def f(x):
  if x:
    return x
  else:
    return f(x+1)
```

Consider the following case; the `else` syntax is redundant as the prior `return` already breaks the flow of the program therefore there is no point for the redundant `else` clause.

```py
def f(x):
  if x:
    return x
  return f(x+1)
```


# when you should be using `if not x`, `if x`, `if x == 1`, `if x == 0`, `if x is None` or `if x == None`

`if not x`: evaluates to True for when:
              `x` -> `0`
              
The condition does not evaluate to True for negatives or any positives -- nor any special float values like `inf` or `nan`
 
 
`if x`: evaluates to True for when:
              `1` <= `x` <= `inf`
              `-inf` <= `x` < `0`
              
The condition does not evaluate to True just for `1`, so keep in mind when doing `if x` (where `x` is arbitrary) that you're not assuming `x` to be `1` (unless within logical reason) and treating it as so.


`if x == 1`: evaluates to True for when:
              `x` -> `1`
              
When doing any form of code-review, don't note something like this as a request for simplification to something like `if x` - since as explained above - unless the conditions exempt this malpractice, you cannot always assume that `x` is `1`, but rather between `-inf, -1 and 1, inf`.


`if x == 0`: evaluates to True for when:
              `x` -> `0`
              
Unlike previous explanations, this can be utmostly simplified to `if not x`.


`if x is None`: evaluates to True for when:
              `x` -> `NoneType`
              
This is the preferred form of checking whether arbitrary `x` is `None` as it is faster (since it doesn't call an underyling `__eq__`).


`if x == None`: evaluates to True for when:
              `x` -> `NoneType`
              
Unlike the relative former example (`if x is None`), this calls the underlying (yet seemingly unexistent/unimplemented-- clarify?) `__eq__` function, which provides call overhead and thus slows it down.


```
 unazed@unazed  ~  python3.7 -m timeit --setup "a = None" -c "a is None" 
10000000 loops, best of 5: 27.9 nsec per loop
 unazed@unazed  ~  python3.7 -m timeit --setup "a = None" -c "a == None" 
10000000 loops, best of 5: 35.9 nsec per loop

 unazed@unazed  ~  python3.6 -m timeit --setup "a = None" -c "a is None"
10000000 loops, best of 3: 0.0245 usec per loop
 unazed@unazed  ~  python3.6 -m timeit --setup "a = None" -c "a == None"
10000000 loops, best of 3: 0.0352 usec per loop

 unazed@unazed  ~  python3.2 -m timeit --setup "a = None" -c "a is None" 
10000000 loops, best of 3: 0.0283 usec per loop               
 unazed@unazed  ~  python3.2 -m timeit --setup "a = None" -c "a == None" 
10000000 loops, best of 3: 0.0362 usec per loop

 unazed@unazed  ~  python2.7 -m timeit --setup "a = None" -c "a is None" 
10000000 loops, best of 3: 0.0326 usec per loop
 unazed@unazed  ~  python2.7 -m timeit --setup "a = None" -c "a == None" 
10000000 loops, best of 3: 0.0547 usec per loop

 unazed@unazed  ~  python2.0
...
>>> start = time.time(); good(None); print("Took: %f seconds" % (time.time()-start))
# GOOD
Took: 0.001744 seconds
>>> start = time.time(); bad(None); print("Took: %f seconds" % (time.time()-start))
# BAD
Took: 0.001774 seconds

 unazed@unazed  ~  python1.6
... 
>>> start = time.time(); good(None); print("Took: %f" % (time.time()-start))
Took: 0.001356
>>> start = time.time(); bad(None); print("Took: %f" % (time.time()-start))
Took: 0.001371

```

Neither Python 3.7.0a2, 3.6.3, 3.2.3, 2.7.14, 2.0.1, 1.6.1 showed any difference for where the `x == None` time was less than `x is None`.


# why (for non-complex functions) you should be returning: `True`, `False` or `None`


By `non-complex functions` I'm referring to functions whose return types are things like class instances or irregular types, however not something like `str`, `int`, `float`, or so forth.

```py
def bad_function(*args, **kwargs):
  if not args:
    return "bad args"
  elif not kwargs:
    return "bad kwargs"
  return "good"
```

Simplified to:

```py
def good_function(*args, **kwargs):
  return bool(args and kwargs)
```

Primitively, the concept applied is returning either `True` or `False` because strings are hard-coded, nasty and only require predefining them at start of code so that you don't make your code extremely static.
`True`, `False` and `None` are all predefined and standard values; they let you implement checks like `if good_function(1, 'oops'):` instead of having to do `if bad_function(1, 'oops') == "good"`. An issue that might arise is when you need more than three return values -- which typically indicates you're doing something wrong and should revise your function -- however, in these cases you can consider options like: a flag byte, raising exceptions, building another type, or returning different integers like 2, -1, 255 etc.


# pure python is faster than regular expressions, so optimize where you can


```py
import re
import sys


pattern = re.compile(r"(\d+)\.(\d+)\.(\d+)\.(\d+)")
match = pattern.match(sys.argv[1])

if not match:
  sys.exit("Failed to match against %r." % sys.argv[1])

for octet in match.groups():
  if not (0 <= int(octet) <= 255):
    sys.exit("Invalid octet %r." % octet)
print("Looks OK.")
```


Here is a simple IP address validator. For 1,000,000 cycles against `127.0.0.1` it took the logic `4.677` seconds to finish, however with a pure Python version:


```py
import sys

try:
  groups = [int(i) for i in sys.argv[1].split('.')]
except ValueError:
  sys.exit("Failed to match.")

for octet in groups:
  if not (0 <= octet <= 255):
    sys.exit("Invalid octet %d." % octet)
print("Looks OK.")
```

Run against 1,000,000 cycles with `127.0.0.1` as input, the code took only `2.7` seconds to run.

`RegEx: Took: 4.677459 seconds`
`Pure: Took: 2.743565 seconds`


# `%` formatting is still faster than `str.format`


```
 unazed@unazed  ~  python3.6 -m timeit -c "'%s %s%c' % ('hello', 'world', '!')"
100000000 loops, best of 3: 0.0164 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -c "'{} {}{}'.format('hello', 'world', '!')"                                                                          
1000000 loops, best of 3: 0.395 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -c "'{0} {1}{2}'.format('hello', 'world', '!')"                                               
1000000 loops, best of 3: 0.431 usec per loop
```

say no more


# `sys.exit`, `os._exit`, `exit`, `quit` and `SystemExit`


`sys.exit`, `exit` and `quit` do all the same underlying task, the former `sys.exit` is preferred however opposed to the other two. However when you don't already have `sys` imported you might instead `raise SystemExit` which doesn't require importing the `sys` library.

`os._exit` is a bit bad, it doesn't call any clean-up procedures unlike the other three choices and just terminates the process.


# `while 1:` instead of `while True:`


In Python 2: `True` is a global variable, so for every reference you make, it equates to a `globals()` variable retrieval which isn't very fast, in Python 3: `True` is a keyword, so it's a built in piece of syntax which doesn't need to be loaded, it's the constant integer equivalent of `1` and so `False` for `0`.

Also, in Python you might notice that integers aren't globals nor locals - they're `CONST`s, so they're faster to load than locals and globals, you might also know (if you've read what I've said above) that `1` is the same as `True`, so you can replace the two and it'll equalize performance over all Python versions 2 to 3.


# implicit better than explicit?


Always remember that `open(...)`'s default access mode is `r`/`read`, so there's no need to go write `open("file", 'r')`- rather - `open("file")`... oh and use context managers.
