bind.py [![Unlicensed work](https://raw.githubusercontent.com/unlicense/unlicense.org/master/static/favicon.png)](https://unlicense.org/)
=======
~~[wheel (GHA via `nightly.link`)](https://nightly.link/KOLANICH-libs/bind.py/workflows/CI/master/bind-0.CI-py3-none-any.whl)~~
~~[![GitHub Actions](https://github.com/KOLANICH-libs/bind.py/workflows/CI/badge.svg)](https://github.com/KOLANICH-libs/bind.py/actions/)~~
![N∅ dependencies](https://shields.io/badge/-N∅_deps!-0F0)
[![Libraries.io Status](https://img.shields.io/librariesio/github/KOLANICH-libs/bind.py.svg)](https://libraries.io/github/KOLANICH-libs/bind.py)
[![Code style: antiflash](https://img.shields.io/badge/code%20style-antiflash-FFF.svg)](https://codeberg.org/KOLANICH-tools/antiflash.py)

**We have moved to https://codeberg.org/KOLANICH-libs/bind.py, grab new versions there.**

Under the disguise of "better security" Micro$oft-owned GitHub has [discriminated users of 1FA passwords](https://github.blog/2023-03-09-raising-the-bar-for-software-security-github-2fa-begins-march-13/) while having commercial interest in success and wide adoption of [FIDO 1FA specifications](https://fidoalliance.org/specifications/download/) and [Windows Hello implementation](https://support.microsoft.com/en-us/windows/passkeys-in-windows-301c8944-5ea2-452b-9886-97e4d2ef4422) which [it promotes as a replacement for passwords](https://github.blog/2023-07-12-introducing-passwordless-authentication-on-github-com/). It will result in dire consequencies and is competely inacceptable, [read why](https://codeberg.org/KOLANICH/Fuck-GuanTEEnomo).

If you don't want to participate in harming yourself, it is recommended to follow the lead and migrate somewhere away of GitHub and Micro$oft. Here is [the list of alternatives and rationales to do it](https://github.com/orgs/community/discussions/49869). If they delete the discussion, there are certain well-known places where you can get a copy of it. [Read why you should also leave GitHub](https://codeberg.org/KOLANICH/Fuck-GuanTEEnomo).

---

This is a tool to allow you to inline constants into functions for fun and profit. In some cases using this tool can give 30% gain of speed. In some, but not in the all ones - [in some cases it actually causes slowdown](https://codeberg.org/KOLANICH-libs/bind.py/issues/4).


How does it work?
-----------------
It just
* moves some closured and global variables into constants, taking attention to arguments
* replaces "LOAD_DEREF" and "LOAD_GLOBAL" opcodes with "LOAD_CONST"
* removes moved variables from function `__closure__` property

Limitations
-----------
* When you define a function, sometimes it uses `LOAD_FAST` opcode to load. For now this case is not taken in account.
* Inlined variables shouldn't be modified because they are marked as constants. Doing this can cause undefined behavior including crashes of interpreter and SIGSEGVs. Adding support for these will require some analysis of control and data flow to check if a variable is modified, spawning a local variable and loading const there. For now the function doesn't check any of such kind of usage, it's your responsibility as a programmer to guarantee this.
* You should inspect bytecode (for example using `dis`) of the func you want to optimize to understand if `inline` could help. If there is no `LOAD_DEREF` and `LOAD_GLOBAL` opcodes in a function it won't help in the current state.
* It doesn't modify functions' text, only bytecode. So source-code showing tools like `??` of IPython won't give accurate information.
* Python can optimize some inlined functions better than this tool, for example
```python
def a():
	return 2+3
```
in fact loads not `2` and `3` but a precomputed `5`, it will have `2` instructions smaller byte-code than
```python
@bind(x=2, y=3)
def a():
	return x+y
```
* Inlining limits hackability. When a variable is inlined it cannot be monkey-patched easily by third-party code.
* This is very beta for now. For now it even breaks itself when inlined. Use it on small functions where there is nothing to break.
* Bytecode is implementation detail of cpython. It can and will change unexpectidly. This will make this to break. It also can be different between implementations.

How to use
----------
* Import the `bind` function
```python
from bind import bind
```

* You can explicitly call the function, providing it the function to optimize and the dict of variables to bind.
```python
c=1
def a(a, b):
	print(a+c, b+d)
a=bind(a, {"c":c, "d":1})
a(1, 2) # 2 3
```
* You can use it as a decorator
```python
c=1
@bind({"c":c, "d":1})
@bind(c=c, d=1) #kwargs-syntax also works
def a(a, b):
	print(a+c, b+d)
a(1, 2) # 2 3
```
* When used as decorator, you can also use kwargs-syntax
```python
@bind(c=1, d=1)
def a(a, b):
	print(a+c, b+d)
a(1, 2) # 2 3
```
* You can specify variables separately
```python
c=1
@bind(c=c)
@bind(d=1)
def a(a, b):
	print(a+c, b+d)
a(1, 2) # 2 3
```
* Inlined variable doesn't depend on closure
```python
c=1
d=1
def a(a, b):
	print(a+c, b+d)
a(1, 2) # 2 3
b=bind(a, {"c":c, "d":1})
d=4
c=4
a(1, 2) # 5 6
b(1, 2) # still 2 3 because c is inlined
```
* Independence on following inlines
```python
b=bind(b, {"c":3, "d":4})
b(1, 2) # still 2 3 because there is no c and d anymore in that func
```
* You can inline all the variables visible from the current scope.
```python
c=1
def a1():
	c=2
	def a():
		print(c)
	c=3
	return a
def a2():
	c=2
	@bind
	def a():
		print(c)
	c=3
	return a
b=a1()
c=a2()
b() # 3
c() # 2
```


Benchmark results
-----------------
To run the benchmark run
```bash
python3 benchmarkGen.py | python3
```
this wil give you the results:
```javascript
{
	"load_global": {
		"orig": 45.899,
		"inlined": 32.968,
		"% faster": 30.347
	},
	"load_deref": {
		"orig": 37.911,
		"inlined": 32.022,
		"% faster": 15.532
	}
}
```
As we see, inlined synthetic functions are really 15-30% faster!


Use case
--------
Situation: you have met some library. You looked into it and saw shitcode.
```python
class B:
	def b1(a):
		print(1+a)
	def b2(a):
		print(2+a)
	...
	def b100500(a):
		print(100500+a)
```.

You have have refactored this shit using
```python
def genBFuncs(lol="lol"):
	for i in range(100500):
		def b(a):
			print(i+a)
		b.__name__="b"+str(i)
		yield b
```

Some folks will say
> It will slow down our lib!

As we know from the benchmark, they are right.

Here is the response:
```python
# you have a function "a" making a functions "b" for some library
def genBFuncs(lol="lol"):
	for i in range(100500):
		@bind(i=i)
		def b(a):
			print(i+a)
		b.__name__="b"+str(i)
		yield b
```

This will rewrite functions bytecode so it will be nearly identical to the original versions.
