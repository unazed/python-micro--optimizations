# python-micro--optimizations
a handful of optimizations, tricks and cool things i've learned while doing python


Just so you know, this only exists for the people that want to micro-optimize code and have the consequence of more unreadable, unmanageable and possibly incompatible code. I don't personally like any of the people that, when you optimize, say *"oh if you're going for speed, why use Python?"* - maybe it just piques my curiosity to make my program as fast as it can be and as efficient sometimes and when people tell you to, in conclusion, learn another language; it pours a huge amount of fuel onto a tenderly fire that was burning in you.

Never should you follow many of these practices in industrial or professional environments; although exceptions can be made, most of the optimizations fall under the categories listed in the former paragraph (more unreadable, unmanageable and possibly incompatible code.


# return statements in single else clauses at the end of functions


```py
def f(x):
  if x:
    return x
  else:
    return f(x+1)
```

Consider the following case; the `else` syntax is redundant as the prior `return` already breaks the flow of the program therefore there is no point for the redundant `else` clause. Surprisingly, even though this is an obvious flaw, I've seen it multiple times so make sure you don't mistakenly make it.

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

**NOTE:** `x == 0` and `not x` have different execution speeds as well:

![](https://i.imgur.com/gcfw7FN.png)

# why (for non-complex functions) you should be returning: `True`, `False` or `None`


By `non-complex functions` I'm not referring to functions whose return types are irregular, non-primitive types like strings, integers, floats etc.; rather referring to functions that *do*/*can* return strings/integers/floats/etc. and it'd suffice it just fine.
For example:

```py
def bad_function(*args, **kwargs):
  if not args:
    return "bad args"
  elif not kwargs:
    return "bad kwargs"
  return "good"
```

Can be simplified to:

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

**NOTE:** I didn't use `map(...)` since (a) it wouldn't catch any errors until converted to a list (b) when it *is* converted to a list, it'll remove the point of any speed provided by the iterator (c) a list comprehension is faster (d) a list comprehension is more Pythonic.


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

**NOTE:** But to make it fair; here's the test results that came back for `f` strings, `str.format` and `% formatting`:
![](https://i.imgur.com/BkSv67f.png)

`% formatting` beats `f` strings roughly by `0.0001` seconds, and all of those beat `str.format` by `0.0002-0.0003`-ish seconds  

# `sys.exit`, `os._exit`, `exit`, `quit` and `SystemExit`


`sys.exit`, `exit` and `quit` do all the same underlying task, the former `sys.exit` is preferred however opposed to the other two. However when you don't already have `sys` imported you might instead `raise SystemExit` which doesn't require importing the `sys` library.

`os._exit` is a bit bad, it doesn't call any clean-up procedures unlike the other three choices and just terminates the process.


# `while 1:` instead of `while True:`


In Python 2: `True` is a global variable, so for every reference you make, it equates to a `globals()` variable retrieval which isn't very fast, in Python 3: `True` is a keyword, so it's a built in piece of syntax which doesn't need to be loaded, it's the constant integer equivalent of `1` and so `False` for `0`.

Also, in Python you might notice that integers aren't globals nor locals - they're `CONST`s, so they're faster to load than locals and globals, you might also know (if you've read what I've said above) that `1` is the same as `True`, so you can replace the two and it'll equalize performance over all Python versions 2 to 3.


# implicit better than explicit?


1) Always remember that `open(...)`'s default access mode is `r`/`read`, so there's no need to go write `open("file", 'r')`- rather - `open("file")`... oh and use context managers.

2) `socket.socket(...)` (as of Python 3) doesn't need to take `family` and `type` parameters, calling `socket.socket()` will automatically create a AF_INET, SOCK_STREAM socket.


# `seq.copy()`, `seq[:]` or `list(seq)`?


Note: `list.copy()` doesn't exist on Python 2

```
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1, 2, 3]" -c "seq.copy()" 
10000000 loops, best of 3: 0.128 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1, 2, 3]" -c "seq[:]"    
10000000 loops, best of 3: 0.112 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1, 2, 3]" -c "list(seq)"
10000000 loops, best of 3: 0.226 usec per loop
```

The literal notation `seq[:]` is faster than `seq.copy()` which are both faster than `list(seq)`, but this is only for a three element list - here follows testing for lists with `256` Python 3.6.3 integers which all have sizes of `48` bytes, therefore making the list approximately, perfectly, 12KB.

```
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1<<160]*256" -c "seq.copy()" 
10000000 loops, best of 3: 1.36 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1<<160]*256" -c "seq[:]"
10000000 loops, best of 3: 1.35 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -n 10000000 --setup "seq = [1<<160]*256" -c "list(seq)"
10000000 loops, best of 3: 1.39 usec per loop
```

The operations which are the fastest follow the same sequence as given for the smaller, three element list above. First `seq[:]`, secondly `seq.copy()` and lastly `list(seq)`.

Ahead of this, there'll be a table for different sized lists with the same `48` byte integer just a different magnitude. Also as a note, the cycles to be run will be smaller else it'll take hours for a single operation to execute; so the cycle size will be `100000`.


| Operation | Amount of Elements | Time Taken |
|-----------|--------------------|------------|
| `seq.copy()` | 512             | 3.08 usec  |
| `seq[:]`     | 512             | 3.06 usec  |
| `list(seq)`  | 512             | 3.05 usec  |
| `seq.copy()` | 1024            | 5.94 usec  |
| `seq[:]`     | 1024            | 5.93 usec  |
| `list(seq)`  | 1024            | 5.85 usec  |
| `seq.copy()` | 2048            | 11.8 usec  |
| `seq[:]`     | 2048            | 11.8 usec  |
| `list(seq)`  | 2048            | 11.7 usec  |
| `seq.copy()` | 4192            | 19.7 usec  |
| `seq[:]`     | 4192            | 19.7 usec  |
| `list(seq)`  | 4192            | 18.6 usec  |
| `seq.copy()` | 8192            | 49.4 usec  |
| `seq[:]`     | 8192            | 50.1 usec  |
| `list(seq)`  | 8192            | 46.9 usec  |
| `seq.copy()` | 16384           | 100 usec   |
| `seq[:]`     | 16384           | 100 usec   |
| `list(seq)`  | 16384           | 93.6 usec  |
| `seq.copy()` | 32768           | 156 usec   |
| `seq[:]`     | 32768           | 156 usec   |
| `list(seq)`  | 32768           | 142 usec   |
| `seq.copy()` | 65536           | 315 usec   |
| `seq[:]`     | 65536           | 313 usec   |
| `list(seq)`  | 65536           | 286 usec   |

Now it's either that I'm drugged or there is actually a speed gain using `list(seq)` on bigger lists but I'm actually perplexed myself about this table, so on doing tests for smaller-sized lists ranging from values between the range of 2 and 256 assuming `log_2 n ∈ Z` with a cycle size of `1000000`.

| Operation | Amount of Elements | Time Taken |
|-----------|--------------------|------------|
| `seq.copy()` | 2               | 0.129 usec |
| `seq[:]`     | 2               | 0.124 usec |
| `list(seq)`  | 2               | 0.228 usec |
| `seq.copy()` | 4               | 0.131 usec |
| `seq[:]`     | 4               | 0.137 usec |
| `list(seq)`  | 4               | 0.236 usec |
| `seq.copy()` | 8               | 0.150 usec |
| `seq[:]`     | 8               | 0.141 usec |
| `list(seq)`  | 8               | 0.237 usec |
| `seq.copy()` | 16              | 0.189 usec |
| `seq[:]`     | 16              | 0.178 usec |
| `list(seq)`  | 16              | 0.271 usec |
| `seq.copy()` | 32              | 0.263 usec |
| `seq[:]`     | 32              | 0.244 usec |
| `list(seq)`  | 32              | 0.382 usec |
| `seq.copy()` | 64              | 0.433 usec |
| `seq[:]`     | 64              | 0.408 usec |
| `list(seq)`  | 64              | 0.518 usec |
| `seq.copy()` | 128             | 0.781 usec |
| `seq[:]`     | 128             | 0.768 usec |
| `list(seq)`  | 128             | 0.835 usec |
| `seq.copy()` | 256             | 1.35 usec  |
| `seq[:]`     | 256             | 1.34 usec  |
| `list(seq)`  | 256             | 1.39 usec  |


So this table agrees with the fact `list(seq)` is slower, so with this one can deduce that `list(seq)` is slower for all lists of magnitude under (thereabout) `1024`.

But I don't want a *rough* assumption, so I'll test for all values between 512 and 1024 non-inclusively. All in steps of `32` because I don't want the table to be humongous.


| Operation | Amount of Elements | Time Taken |
|-----------|--------------------|------------|
| `seq.copy()` | 544             | 2.65 usec  |
| `seq[:]`     | 544             | 2.65 usec  |
| `list(seq)`  | 544             | 2.65 usec  |
| `seq.copy()` | 576             | 2.79 usec  |
| `seq[:]`     | 576             | 2.80 usec  |
| `list(seq)`  | 576             | 2.77 usec  |
| `seq.copy()` | 608             | 2.94 usec  |
| `seq[:]`     | 608             | 3.00 usec  |
| `list(seq)`  | 608             | 2.92 usec  |
| `seq.copy()` | 640             | 3.12 usec  |
| `seq[:]`     | 640             | 3.11 usec  |
| `list(seq)`  | 640             | 3.05 usec  |
| `seq.copy()` | 672             | 3.24 usec  |
| `seq[:]`     | 672             | 3.27 usec  |
| `list(seq)`  | 672             | 3.21 usec  |
| `seq.copy()` | 704             | 3.42 usec  |
| `seq[:]`     | 704             | 3.38 usec  |
| `list(seq)`  | 704             | 3.36 usec  |


Now, I cut the list early because (a) I'm not going to wait 20 minutes for the rest of it to complete and (b) I've found where the `list(seq)` speed intersects the other two operations' speeds.
As you can see, for the first three records, `seq.copy()` and `seq[:]` draw in speed and `list(seq)` is 0.01 usec faster, then for the next record: `seq.copy()` is second, `seq{:]` is last, and `list(seq)` overtakes first, then finally the third record shows `seq.copy()` coming second again, `seq[:]` coming last and `list(seq)` staying in first for speed.

**CONCLUSION:**

`seq.copy()` for all sequences with magnitude below 544, will be second fastest to `seq[:]` and the slowest method will be `list(seq)`; however, for all sequences with a bigger magnitude, `list(seq)` will be fastest and both `seq[:]`and `seq.copy()` will have a seemingly random disparity. 

*NOTE:* Element size doesn't seem to affect the operations' speed.


# should i use json, xml, yaml, csv or create my own format?


This presents many of the same issues that you would deal with when you're creating another form of implementation for something already standardized in many ways. Firstly, you're going to make many, many mistakes in the design and most likely not consider the many external use-cases it could propose, thus making the implementation worthless as a whole because unless you can find other places where the certain design can be integrated without any tweaking -- there's no generality therefore there's no real reusability.

JSON is a versatile, nice-to-read and flexible notation. It has the direct relation with Python thereby that the `dict` type follows a more open-ended support for objects.

XML is a markup language which is just a tad bit more explicit than something like JSON; but in my personal opinion it looks a bit bloated and monotone with the tags and it's less readable than JSON. Python has a third-party library for parsing XML, but you'll have to download it from PyPI.

YAML is a greatly user-friendly format which isn't really as common as any of the other two formats, but it's still comparatively different.

CSV is just a BTEC way of doing any of the above in a lazy fashion by just separating values (that hopefully don't have commas already) with a comma delimiter. Although as this format is very simple, it is as well extensible to different delimiters like perhaps `\x00` or `\xFF` for user data.

For your own format, as I've said in the first part of my explanation; you have to make sure that your format can be applied anywhere without any possible incompatibilities, also you should never tie a metaphorical knot on it; if you leave it open-ended it'll be able to be put under one's completely different interpretation thus customizable and plugged into a different environment. 
You can see this in place for the four formats listed above as JSON, it's simple and maps `x: ` to `y` where `x` is typically hashable.
In XML, it's bare-bone but with the implementation of the human - it allows for creating theoretical contexts with a really simple system. But I can't deny it's not much more extensible than JSON.
YAML and CSV both share the above traits interlinking JSON and XML, thus it should show a common pattern between these widely used notations. And should show how notations are generalized.

The only different thing about all of these ways of displaying information, is the way that it's written. Some are nicer to look at, some are nicer to write and some are nicer to parse.


# len(list) == x


A faster alternative is to do `list[x-1:]`, because of how Python handles slices, this won't cause an error like `list[x-1]` would (the `-1` is for zero-indexing):

```
 unazed@unazed  ~  python3.6 -m timeit -n 20000000 --setup "a = [1, 2, 3]" -c "a[3:]"                                             
20000000 loops, best of 3: 0.0992 usec per loop
 unazed@unazed  ~  python3.6 -m timeit -n 20000000 --setup "a = [1, 2, 3]" -c "len(a) == 4"
20000000 loops, best of 3: 0.105 usec per loop
```


# another way of getting the last element in a list


So, there's two [three] main ways of getting the last element in a list:
- `list[-1]`
- for when `list = [1, 2, 3]`, `list[2]` (slower than `list[-1]`?)
- `list(reversed(list))[0]` (/s)

But, believe it or not, there's a more case-specific and yet faster way of getting the last item without having to call *any* underlying/hidden nor explicit functions.

```py
a = [...]

for i in a:
  ...
last_item = i
```

Logical enough, right? Python doesn't have block scopes, so this'd work as intended.


# does listening to music whilst coding help in any way/shape or form?


Subjectively, it doesn't help me if the music has vocals or something I have to spend time processing and analyzing ephemerally; I miss things out, I make syntax errors and my logic becomes volatile. But I do enjoy a quiet and light tranquil piano melody playing during when I code - but even then I still need my piece of mind when I'm reading things else I'll be distracted even by a single piano note.

Sadly, there's no general answer, I can't tell you that rock music will make your code any worse than somebody else's code would be when they're listening to ASMR; it's all based around how your brain processes things and how you partition your attention. 
Not going to lie though, those `programming mix`xes are just a tad bit weird.

Cliché-ly, the answer is going to end up being that your own experimentation will lead to your best fit.


# should i use raw sockets or should i create my own interface to raw sockets?


Using sockets is cool, but so is using sockets and making your own interface. Repetitive code isn't very nice, but the overhead with the function calls isn't efficient. Shorter, concise and readable code is beautiful - but having ability to maximize your code's utility is always nice.

Speaking perfomance-wise, an OOP interface will *always* be slower than the raw, unabstracted interface to the library. When you're creating an interface the only things that you're preventing are semantical mistakes, but all you're really doing is squeezing a bunch of liquids into a container. Sure, they'll stay consistent wherever you pour them; no mistake about that, but any mistakes in the actual 'squeezing' bit, perhaps missing some functionality or failing to generalize it; then you'll just mix one container of incompatible liquid with some reactive surface which'll create a huge mess and practically create the inverse effect that you want to actually achieve.

Raw sockets are faster and more customizable; but their prologues are repeated so many times and equally so their epilogues.


```py
import socket
import sys


sockfd = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sockfd.settimeout(2)
try:
  sockfd.connect((host, port))
except socket.timeout:
  print("couldn't connect to %s:%s" % (host, port))
  sys.exit(1)
...
sockfd.close()
```


6 references to `socket`, one import from `sys`, 4 references to `sockfd` and the overhead of `try: ... except (,): ...`.


```py
# newsocket.py

import socket


class Socket(object):
  def __init__(self, host, port, connect=True, timeout=2):
    self.host = host
    self.ip = ip
    self.timeout = timeout
    self._socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    if connect:
      self.connect()
      
  def connect(self):
    self._socket.settimeout(self.timeout)
    try:
      self._socket.connect((self.host, int(self.ip)))
    except (socket.timeout, ValueError):
      return False
    return True
  
  def close(self):
    self._socket.close()
    

# main.py

from newsocket import Socket

sockfd = Socket(host, port, False)

if sockfd.connect():
  print("Connected")
else:
  print("couldn't connect to %s:%s" % (host, port))
  
sockfd.close()
```

2 references to `Socket`, 3 references to `sockfd` and not a single `sys.exit`. A drastic improvement from the linear paradigm shown in the other example.
But, if you have any intellect you'd notice that, well clearly there's waaay more references to `Socket` and there is still a `try: ... except (,): ...` embedded within, just not seen in `main.py`. Well that's the point of an interface and that displays the only real features that ir provides, an interface will never be faster, besides fixing any possible mistkaes that could've been made in the task of rewriting code.


# list conversion. best methods?

So there's a few ways of converting non-`list` types into lists either for mutability - or, space-consumption:

- `[*tuple]`
- `[i for i in tuple]`
- `list(tuple)`

Here are results for all of them with differing list magnitudes:

![](https://i.imgur.com/RIokUAY.png)

Surprisingly, `[i for i in tuple]` was the fastest method of conversion even though it creates a redundant variable, `i` (which is, still, freed.) Also I had a little hunch in my head that the shortest way would be the fastest, however, `[*tuple]` seems to be on the same line of growth as `list(tuple)` is, albeit a tiny bit slower.
