12 years of Pylint
==================

.. class:: title

-----------------


.. class:: center

    .. class:: title

       12 years of Pylint   

    |
    |
    |

    **Claudiu Popa**

    .. epigraph::

        `claudiu@ropython.org <claudiu@ropython.org>`_

        `bitbucket.org/pcmanticore <http://bitbucket.org/pcmanticore>`_
        

Presenter Notes
---------------

* Widespreaded
* History, detour through static analysis
        

-----

What is this *lint* you're talking about?
=========================================

* lint is a program which analyses your code, looking for errors.

* you can use it as a verb: ``Imma gonna lint your code``

* **Pylint** is more than a linter though:

  * a style checker, enforcing PEP 8 rules
  
  * a linter, looking for suspicious code
  
  * a type checker and structural analyser

-------


Pylinting ugly code goes like this
==================================

.. code-block:: python

    import os, sys
    from csv import reader

    def double(x ):
        return (x  
                   + x)

    def square(x):
        return x*x

----


Pylinting ugly code goes like this
==================================

.. code-block:: sh

    $ pylint a.py

    C:  4, 0: No space allowed before bracket
    def double(x ):
                 ^ (bad-whitespace)
    C:  5, 0: Trailing whitespace (trailing-whitespace)
    C:  6, 0: Wrong continued indentation (remove 3 spaces).
                   + x)
                |  ^ (bad-continuation)
    C:  4, 0: Invalid argument name "x" (invalid-name)
    C:  4, 0: Missing function docstring (missing-docstring)
    C:  8, 0: Invalid argument name "x" (invalid-name)
    W:  1, 0: Unused import os (unused-import)
    W:  1, 0: Unused import sys (unused-import)
    W:  2, 0: Unused reader imported from csv (unused-import)
    
----

Pylint can detect real bugs too
===============================

* using undefined variables

* accessing undefined members

* calling non-callable objects

   .. sourcecode:: python

      $ cat a.py

      import zipfile
      f = zipfile.ZipFile(outfile, 'w', zipfile.DEFLATED)
      f()
   
      $ pylint a.py
   
      E:  2,20: Undefined variable 'outfile' (undefined-variable)
      E:  2,42: Module 'zipfile' has no 'DEFLATED' member (no-member)
      E:  3, 1: f is not callable (not-callable)

-----


Pylint can detect real bugs too
===============================

* special methods implemented incorrectly

   .. sourcecode:: python

      $ cat a.py
    
      class MyContextManager(object):
          def __enter__(self):
              pass
            
          # It needs three arguments      
          def __exit__(self):
              pass
                
      $ pylint a.py

      E: The special method '__exit__' expects 3 params, 0 was given
   
-----


Pylint can detect real bugs too
===============================

* constant if conditions

    .. code-block:: python

       $ cat a.py
     
       def func():
           return bool(some_condition)
       
       # func is always true   
       if func:
           pass
           
       $ pylint a.py

       W:  5: Using a conditional statement with a constant value

------

Pylint can detect real bugs too
===============================

* try to figure out what's the problem in this code.

* should print 1, 2, 3, 4, ..., 9 right?

   .. sourcecode:: python

       def bad_case2():
           return [(lambda: i) for i in range(10)]

       for callable in bad_case2():
           print(callable())



-------

Pylint can detect real bugs too
===============================

* actually no:

   .. sourcecode:: python
  
      $ python a.py
      9
      9
      9
      ...

      $ pylint a.py
      W:  2,20: Cell variable i defined in loop

* the previous code created a closure and i was looked up
  in the parent's scope when executed.

* **i** in the parent's scope after the loop was 9.


------

   
   
   
12 years of what?
=================

* one of the oldest (maintained) static analysis tool
* created by Logilab (Sylvain Thenault) in 2003
* Google uses its own version internally: gpylint
* over 35000 lines of code + tests, according to ohloh.net

   * pylint: 2416 commits, 21536 lines of code
   * astroid: 1604 commits, 14045 lines of code

----


Pylint's new life
=================

* My first patch was accepted in Pylint 1.0 - 2013
* Commit rights gained in Pylint 1.1 - 2013
* Maintainer since Pylint 1.2 - 2014
* The only active maintainer since Pylint 1.3 - 2014
* Pylint 2.0 in 2016

----


Static analysis
===============

* analysing of a computer software without executing programs

* you can benefit from using static analysis if:

   * running tests takes a lot of time or work
   
   * you don't have tests for a legacy system
   
   * you need a form of automatic reviews
   
* not equivalent to a review


----------

How pylint works?
=================

* there's a split between the verifications (pylint) and the component that understands
  Python (astroid)

* follows the general pattern of building a linter: uses ASTs

* ASTs - abstract syntax trees - are a intermediate representation between code and bytecode

* They are a structural and expressive form of holding information


--------

How pylint works?
=================

* We use the Python ``ast`` module internally

  .. sourcecode:: python
  
     $ cat a.py
   
     from ast import parse, dump
     module = parse('''
     def test(a, b, *, foo=None):
         pass
     ''')
     print(dump(module))
   
------


How pylint works?
=================

* ast module is great, but it is not backwards compatible

* astroid strives to be a compatibile layer that between various new versions of **ast**

* it has a similar API with the **ast** module

  .. sourcecode:: python

     from astroid import parse
     module = parse('''
     def test(a, b, *, foo=None):
          pass
     ''')
     print(module.repr_tree())
			
------

Astroid nodes
=============

* the nodes are almost equivalent with the one from the ast module

  * `CallFunc` - function call

  * `Function` - function definition

  * `Class` - a class definition

  * `Arguments` - a function's arguments

  * etc

------

Astroid nodes
=============

* astroid nodes provide useful capabilities

  * you can get a node's parent:

    .. sourcecode:: python

       >>> from astroid import extract_node
       >>> node = extract_node('''f = 42''')
       >>> node
       <Assign() l.2 [] at 0x2c49dd0>
       >>> node.parent
       <Module() l.0 [] at 0x2c49d90>

-----

Astroid nodes
=============

* you can get the children of a node

  .. sourcecode:: python


       >>> node = extract_node('''
           def test():
              europython = 1
              foo = 42
           ''')
       >>> list(node.get_children())
       [<Arguments() l.2 [] at 0x2bb2114208>,
        <Assign() l.3 [] at 0x2bb2114278>,
        <Assign() l.4 [] at 0x2bb2114320>]

----

Astroid nodes
=============

* you can get a node's lexical scope

    .. sourcecode:: python

       >>> node = extract_node('a = 1')
       >>> node.scope()
       <Module() l.0 [] at 0x2c49d90>
       >>> node = extract_node('''
           def test():
               foo = 42 #@
           ''')
       >>> node.scope()
       <Function(test) l.2 [] at 0x2bfbf10>
       >>> node = extract_node("[__(i) for i in range(10)]")
       >>> node.scope()
       <ListComp() l.2 [] at 0x795684240>
   
----

Astroid nodes
=============


* you can get a node's locals

    .. sourcecode:: python

       >>> module.locals
       {'f': [<AssName(f) l.2 [] at 0xd1b6191748>]}

* or a node's string representations (roundtrips back to the original code)

    .. sourcecode:: python

       >>> module.as_string()
       'f = 42'

----

Astroid nodes
=============

* some nodes are augmented with capabilities tailored for them

  .. sourcecode:: python

     klass = extract_node('''
     from collections import OrderedDict
     class A(object): pass
     class B(object): pass
     class C(A, B): object
     class OmgMetaclasses(OrderedDict, C, metaclass=abc.ABCMeta):
         __slots__ = ('foo', 'bar')
         version = 1.0
     ''')

-----

Astroid nodes
=============

* getting a class's slots

  .. sourcecode:: python

     >>> klass.slots()
     [<Const(str) l.4 [] at ...>, <Const(str) l.4 [] at ...>]

* getting a class's metaclass

  .. sourcecode:: python

      >>> klass.metaclass()
      <Class(ABCMeta) l.109 [abc] at 0x9cfd5e6470>

* getting a class's method resolution order

  .. sourcecode:: python

  >>> klass.mro()
  [<Class(OmgMetaclasses) l.8 [] at ...>,
   <Class(OrderedDict) l.43 [collections] at ...>,
   <Class(dict) l.0 [builtins] at ...>, <Class(C) l.6 [] at ...>,
   <Class(A) l.4 [] at ...>, <Class(B) l.5 [] at ...>,
   <Class(object) l.0 [builtins] at ...>]

-----

Astroid nodes - inference
=========================

* the critical ability that astroid nodes have is to do *inference*

* inferring is the act of resolving what a node really is

* similar with type inference, but we are more interested in what a node
  really represents, rather than its type value

* each node type provides its own inference rules, according to Python's semantics

* the inference also does partial abstract interpretation

  * we evaluate what the side effect of a statement will actually be

----

Astroid nodes - inference example
=================================

.. sourcecode:: python


  n = extract_node('''
  def func(arg):
    return arg + arg

  func(24)
  ''')
  
  >>> n
  CallFunc() l.5 [] at 0x6360d01b00>
  >>> inferred = next(n.infer())
  <Const(int) l.None [int] at 0x94764b1908>
  >>> inferred.value
  48

----

Astroid nodes - inference example
=================================

.. sourcecode:: python

  class A(object):
      def __init__(self):
          self.foo = 42
      def __add__(self, other):
          return other.bar + self.foo / 2
  class B(A):
      def __init__(self):
          self.bar = 24
      def __radd__(self, other): return NotImplemented
  A() + B()
  
  >>> n
  <BinOp() l.12 [] at 0x66d4e9ce80>
  >>> inferred = next(n.infer())
  >>> inferred.value
  45.0

-------


Astroid nodes - transforms
==========================

* we can't possibly understand everything (try to understand namedtuple for instance)

* we provide an API for transforming parts of the tree, by changing each node
  with the result from a transform function

* we already use this API for understanding namedtuples, enums, six.moves etc.

------

Astroid nodes - transforms
==========================

* the transform is a function that receives a node and
  returns the same node modified or a completely new node

* they need to be registered using an internal manager

  .. sourcecode:: python

     def transform_six_add_metaclass(node):
        ...

     MANAGER.register_transform(nodes.Class, transform_six_add_metaclass,
                                looks_like_six_add_metaclass)

* you can filter the nodes you want to be transformed by using a filter function

-----

Astroid nodes - inference tips
==============================

* we also provide a way to add new inference rules

* we already use this API for understanding builtins: super, type, isinstance, callable, list, frozenset etc

  .. sourcecode:: python

     def infer_super(node):
          # Return an iterator of results
         return iter(inference_results)

     MANAGER.register_transform(nodes.CallFunc,
                                inference_tip(infer_super))

-----

Astroid capabilities
====================

* having good inference improves the linter.

* We understand:

  * super, the method resolution order of your classes

  * isinstance, issubclass, getattr, hasattr, type

  * binary arithmetic operations, logical operators, comparisons

  * context managers

  * list, dict, tuple, string indexing and slicing

-----

Astroid capabilities
====================

.. sourcecode:: python

  class A(object):
    def spam(self): return "A"
    foo = 42
    @staticmethod
    def static(arg): print(arg)

   class B(A):
    def boo(self, a): print(a)

   class C(A):
    def boo(self, a, b): print(a, b)

   class E(C, B):
    def __init__(self):
        super(E, self).boo(4, 5) 
        super(C, self).boo(5, 6)
        super(E, self).foo()
        super(E, self).static()
        super(E, self).spa

----

Astroid capabilities
====================

.. sourcecode:: python

    $ pylint a.py ...
    E: 16,26: Too many positional arguments for method call
    E: 17,26: super(E, self).foo is not callable
    E: 18,29: No value for argument 'arg' in staticmethod call
    E: 19,23: Super of 'E' has no 'spa' member

-----

Astroid capabilities
====================

.. sourcecode:: python

   import contextlib
 
   def real_func():
      pass

   class A:
      @contextlib.contextmanager
      def meth(self):
         yield real_func

   classes = [A()]
   a = classes[0]
   meth = hasattr(a, 'meth') and callable(a.meth) and getattr(a, 'meth')
   with meth(42) as foo:
       foo('EuroPython is great')   

   $ pylint a.py ...
   E: Too many positional arguments for method call

----- 


Pylint
======

* pylint is a fancy walker over the tree provided by astroid

* it uses the visitor pattern to walk the tree

* on each visited node, it checks to see if there is any rule that it should verify against

.. sourcecode:: python

   class TypeChecker(BaseChecker):

       def visit_getattr(self, node):
           ...
       def visit_callfunc(self, node):
           ...

-----


Pylint - checkers
=================

* We have multiple checkers, each trying to detect a particular type of error

* TODoooooooo


------------------------


Pylint
======

* comes with a lot of goodies and it has a vibrant ecosystem:

* you can write your own checker, even though that implies some knowledge of Python and how pylint works

* plenty of additional packages tailored for specific frameworks:
  pylint-flask, pylint-django, pylint-celery, pylint-fields

* run your checker as this:

  .. code-block:: python

     $ pylint a.py --load-plugins=plugin a.py

-----

Pylint - Pyreverse
==================

* get UML diagrams from your packages

* Graphviz must be installed in order to work properly


  .. code-block:: python

     $ pyreverse -o png pylint

.. image:: pylint_uml.png
    :class: white center

------

Pylint - Spellchecking
======================

* spell check your comments and docstrings (needs python-enchant to be installed)


   .. code-block:: python
   
      $ pylint --spelling-dict=en_US a.py
      C:  1, 0: Wrong spelling of a word 'speling' in a docstring:
      Verify that the speling cheker work as expcted.
                      ^^^^^^^
      Did you mean: 'spieling' or 'spelling' or 'spelunking'?

--------------


Pylint
======

* My favourite is the Python 3 porting checker

* Also recommended by the official HowTo porting guide: https://docs.python.org/3/howto/pyporting.html

* detects

  * using removed syntax: print statement, old raise form, parameter unpacking
  * using removed builtins: apply, cmp, execfile etc
  * using removed special methods: __coerce__, __delslice__ etc
  * assigning to __metaclass__ attribute
  * using map / filter / reduce in non iterating context

-----

Pylint - Python 3 porting checker 
=================================

.. code-block:: sh

    def func(op):
        # do some serious operation with op

    # Not evaluating, func will never be called
    map(func, entries)   

    print 'lala'
    b =basestring()
    f = cmp(2, 3, 4)

    class A:
        __metaclass__ = type
        def __setslice__(self, other):
            pass

    raise Exception, "lala"

------

Pylint - Python 3 porting checker
=================================

.. code-block:: sh

  $ pylint a.py --py3k

  W:  5, 0: map built-in referenced when not iterating
  E:  7, 0: print statement used
  W:  8, 3: basestring built-in referenced
  W:  9, 4: cmp built-in referenced
  W: 11, 0: Assigning to a class' __metaclass__ attribute
  W: 13, 8: __setslice__ method defined
  E: 16, 0: Use raise ErrorClass(args) instead of raise ErrorClass, args.

----


Similar tools: pyflakes
=======================
 

* pyflakes: lightweight, fast, but detects only handful of errors

   .. code-block:: python

       def test():
           a, b = [1, 2, 3] # unbalanced tuple unpacking
           try:
               if None: # constant check
                   pass
           except True: # catching non exception
               pass

      $ pyflakes a.py
      a.py:2: local variable 'a' is assigned to but never used
      a.py:2: local variable 'b' is assigned to but never used   

-----

Similar tools: Pychecker
========================
  
* pychecker: forefather of Pylint, not really static, ahead of its time, now dead

* still detects issues that most of static analysers don't detect

   .. code-block:: python

      $ pychecker a.py
   
      a.py:2: Unpacking 3 values into 2 variables
      a.py:4: Using a conditional statement with a constant value
      a.py:6: Catching a non-Exception object (True)

-------


Similar tools: jedi and mypy
============================

* jedi: autocompletion library, wants to be a static analyser, a lot of hardcoded behaviour

   .. code-block:: python

       $ python -m jedi linter a.py
       $ # it detected nothing :(

* mypy: cool, Guido loves it, PEP 484 started from here. The real competitor of pylint.

   .. code-block:: python

     $ mypy a.py
     a.py: In function "test":
     a.py,line 2: Too many values to unpack (2 expected, 3 provided)
   
------

Static analysis shortcomings
============================

* you can't fully understand code when

   * dynamic code is invoked

   * extension modules are involved

   * you don't understand flow control

   * the code you're supposed to understand is too smart.

--------------

Static analysis shortcomings
============================

* nose

  .. code-block:: python

     for at in [ at for at in dir(_t)
                if at.startswith('assert') and not '_' in at ]:
       pepd = pep8(at)
       vars()[pepd] = getattr(_t, at)
       __all__.append(pepd)

* multiprocessing

  .. code-block:: python

     globals().update((name, getattr(context._default_context, name))
                       for name in context._default_context.__all__)
      __all__ = context._default_context.__all__ 
   
-------

Future Pylint
=============

* converges towards Pylint 2.0

* full flow control analysis

* a better data model (undestanding descriptors, proper attribute access logic)

* support for PEP 484 and stub files

* better abstract interpretation and evaluation

* bringing more contributors into the project


---------

.. class:: center

    .. class:: title

    Thank you!

