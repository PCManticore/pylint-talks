﻿
=========

.. class:: center

    .. class:: title

       **12 years of Pylint**
       
           or

       *How I Learned to Stop Worrying and Love the bugs*

    |
    |
    |

    **Claudiu Popa**

    .. epigraph::

        `bitbucket.org/pcmanticore <http://bitbucket.org/pcmanticore>`_

        `@PCManticore <http://twitter.com/PCManticore>`_

        

-----

What is this *lint* you're talking about?
=========================================

* lint is a program which analyses your code, looking for errors.

* you can use it as a verb: ``I'm gonna lint your code``

* **Pylint** is more than a linter though:

  * a style checker, enforcing PEP 8 rules
  
  * a linter, looking for suspicious code
  
  * a type checker and structural analyzer


Presenter Notes
---------------

* Widespreaded
* More than a linter
* Over 180 verifications

-------

Static analysis
===============

* analysing of a computer software without executing programs

* you can benefit from using static analysis if:

   * running tests takes a lot of time or work
   
   * you don't have tests for a legacy system
   
   * you need a form of automatic reviews
   
* not equivalent to a review


Presenter Notes
---------------

* static program analysis is the analysis of computer software
  that is performed without actually executing programs,
  having the purpose of finding bugs and other issues in our code.

----------


Pylinting ugly code goes like this
==================================

.. code-block:: python
   :linenos:

   import os

   def process_stuff(params):
      executed = False
      if not params:
         raise ValueError('empty command list')
         # I didn't intended to put this here.
         for foo in params:
            # Oups, forgot to call it
            foo.execute

.. code-block:: sh

    $ pylint a.py

    W:  8: Unreachable code
    W: 10: Statement seems to have no effect
    W:  4: Unused variable 'executed'
    W:  1: Unused import os

Presenter Notes
---------------

* examples of unreachable code

* statements without effects (unintended)

* unused variables and imports
    
----

Pylint can detect real bugs too
===============================

* using undefined variables

* accessing undefined members

* calling objects which aren't callable

   .. sourcecode:: python

      $ cat a.py

      import zipfile
      f = zipfile.ZipFile(outfile, 'w', zipfile.DEFLATED)
      f()
   
      $ pylint a.py
   
      E:  2,20: Undefined variable 'outfile' (undefined-variable)
      E:  2,42: Module 'zipfile' has no 'DEFLATED' member (no-member)
      E:  3, 1: f is not callable (not-callable)

Presenter Notes
---------------

* other real bugs: undefined variables

* undefined members

* calling something which is not callable

-----


Pylint can detect real bugs too
===============================

* or special methods implemented incorrectly

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

Presenter Notes
---------------

* special methods analyzer: dunder exit requires 3 parameters
   
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

Presenter Notes
---------------

* constant checks, might hide bugs

* func was intended to be called here

------

Pylint can detect real bugs too
===============================

* try to figure out what's the problem in this code.

* should print 1, 2, 3, 4, ..., 9 right?

   .. sourcecode:: python

       def bad_case2():
           return [(lambda: var) for var in range(10)]

       for callable in bad_case2():
           print(callable())


Presenter Notes
---------------


* take a couple of moments to detect what's wrong
  with this code

* take a sip of water



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
      W:  2,20: Cell variable 'var' defined in loop

* the previous code created a closure and **var** was looked up
  in the parent's scope when executed.

* **var** in the parent's scope after the loop was 9.


------

   
   
   
12 years of what?
=================

* one of the oldest (maintained) static analysis tool
* created by Logilab (Sylvain Thenault) in 2003
* Google uses its own version internally: gpylint
* over 35000 lines of code + tests, according to ohloh.net

   * pylint: 2416 commits, 21536 lines of code
   * astroid: 1604 commits, 14045 lines of code

* GPL licensed :-(


Presenter Notes
---------------

* oldest, still maintained

* community driven

* the project has little involvement from Logilab

* GPL was really popular in 2003  

----


Pylint's new life
=================

* My first patch was accepted in Pylint 1.0 - 2013
* Commit rights gained in Pylint 1.1 - 2013
* Maintainer since Pylint 1.2 - 2014
* The only active maintainer since Pylint 1.3 - 2014
* Pylint 2.0 in 2016

  * abstract interpretation

  * control flow graphs

  * PEP 484

----


How pylint works?
=================

* there's a split between the verifications (pylint) and the component that understands
  Python (astroid)

* follows the general pattern of building a linter: uses ASTs

* ASTs - abstract syntax trees - tree representation of the sintactic structure
  of source code

* uses the **ast** module internally

  .. sourcecode:: python
   
     from ast import parse, dump
     module = parse('''
     def test(a, b, *, foo=None):
         pass
     ''')
     print(dump(module))
   
------

How pylint works?
=================

.. class:: center

   .. image:: ast.svg
      :width: 650
      :height: 650

.. rubric:: Footnotes

.. [#f1] http://hackflow.com/blog/2015/03/29/metaprogramming-beyond-decency/


-----

How pylint works?
=================

* ast module is great, but it is not backwards compatible

* astroid strives to be a compatibile layer between various new versions of **ast**

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

* astroid nodes provide useful capabilities

  * you can get a node's parent:

    .. sourcecode:: python

       >>> from astroid import extract_node
       >>> node = extract_node('''
       def func():
           f = 42 #@
       ''')
       >>> node
       <Assign() l.2 [] at 0x2c49dd0>
       >>> node.parent
       <Function() 1.2 [] at 0x2c49d80>
       >>> node.parent.parent
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
       >>> node = extract_node("[__(foo) for foo in range(10)]")
       >>> node.scope()
       <ListComp() l.2 [] at 0x795684240>


Presenter Notes
---------------

* On Python 3 the scope of the `foo` is the list comprehension itself

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

Inference
=========

* the critical ability that astroid nodes have is to do *inference*

* inferring is the act of resolving what a node really is

* each node type provides its own inference rules, according to Python's semantics

* the inference also does partial abstract interpretation

  * we evaluate what the side effect of a statement will actually be

Presenter Notes
---------------

* similar with type inference, but we are more interested in what a node
  really represents, rather than its type value

----

Inference example #1
====================

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

Inference example #2
====================

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

Presenter Notes
---------------

* LHS supertype + RHS subtype
* rhs.__radd__ first
* lhs.__add__ second

-------


Node transforms
===============

* we can't possibly understand everything (try to understand namedtuple for instance)

* we provide an API for transforming parts of the tree, by changing each node
  with the result from a transform function

* we already use this API for understanding namedtuples, enums, six.moves etc.

------

Node transforms
===============

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

Inference custom rules
======================

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
   :linenos:

    class A(object):
       def spam(self): return "A"
       foo = 42

    class B(A):
       def boo(self, a): print(a)

    class C(A):
       def boo(self, a, b): print(a, b)

    class E(C, B):
       def __init__(self):
          super(E, self).boo(4, 5) 
          super(C, self).boo(5, 6)
          super(E, self).foo()
          super(E, self).spa

Presenter Notes
---------------

* take a couple of moments to find the bugs from this code

----

Astroid capabilities
====================

* Since astroid knows how super works and understands
  the method resolution order, pylint can detect the errors
  from the previous code

  .. sourcecode:: python

     $ pylint a.py ...
     E: 14,26: Too many positional arguments for method call
     E: 15,26: super(E, self).foo is not callable
     E: 16,23: Super of 'E' has no 'spa' member

-----

Astroid capabilities
====================

.. sourcecode:: python

   def real_func():
      pass

   class A:
      @contextlib.contextmanager
      def meth(self):
         yield real_func

   a = [A(), 1, 2, 3][0]
   meth = hasattr(a, 'meth') and callable(a.meth) and getattr(a, 'meth')
   with meth() as foo:
       foo('EuroPython is great')   

   $ pylint a.py ...
   E: Too many positional arguments for method call


Presenter Notes
---------------

* a more complex example: list indexing, hasattr, callable, getattr

* boolean operators, context managers

----- 


Pylint - patterns over AST nodes
================================

* pylint is a fancy walker over the tree provided by astroid

* the verifications can be seen as patterns that are applied to certain nodes

* it uses the visitor pattern to walk the tree

   .. sourcecode:: python

       class TypeChecker(BaseChecker):
       
          def visit_callfunc(self, node):
             ...

-----

Pylint - visitor pattern example
================================

.. sourcecode:: python

   import collections; print(collections.default)


* **visit_getattr** is called with **Getattr(expr=Name(id='collections'), attrname='defaultdict')**
  as argument

* **node.expr**, which is a Name node, is inferred in order to obtain the **Module** node

* Check if **Module.getattr(node.attrname)** raises NotFoundError

* Apply post-failure filters: owner is a class with unknown base classes, mixin class etc.

----

Pylint - abstract interpretation
================================

* we're using inference, but that doesn't help when having multiple lines of code
  modifying the same object

* they need to be **interpreted** somehow. See this example for instance,
  no way to reason if the current instance has the attribute from line 5 

  .. sourcecode:: python
     :linenos:

     def __init__(self, **kwargs):
         self.__dict__.update(kwargs)

     def some_other_method(self):
         return self.some_arguments_set_in_dunder_init()

----

Pylint - checkers
=================

* We have multiple categories of errors we can detect

  * conventions (PEP 8 mostly)

  * refactorings (circular import dependencies)

  * warnings (code which is not guaranteed to be a bug)

  * errors (most likely bugs in user application)

* Two types of checkers: AST based and token based


------------------------


Pylint
======

* comes with a lot of goodies and it has a vibrant ecosystem

* you can write your own checker, even though that implies some knowledge of Python and how pylint works

* plenty of additional packages tailored for specific frameworks:
  pylint-flask, pylint-django, pylint-celery, pylint-fields

* run your checker as this:

  .. code-block:: python

     $ pylint --load-plugins=plugin a.py

-----

Pylint - extra features
=======================

* pyreverse - generate UML diagrams for your project

* spell check your comments and docstrings (needs python-enchant to be installed)


   .. code-block:: python
   
      $ pylint --spelling-dict=en_US a.py
      C:  1, 0: Wrong spelling of a word 'speling' in a docstring:
      Verify that the speling cheker work as expcted.
                      ^^^^^^^
      Did you mean: 'spieling' or 'spelling' or 'spelunking'?

* Python 3 porting checker

--------------


Pylint - Python 3 porting checker
=================================

* My favourite is the Python 3 porting checker

* Also recommended by the official HowTo porting guide: https://docs.python.org/3/howto/pyporting.html

* can detect:

  * using removed syntax: print statement, old raise form, parameter unpacking
  * using removed builtins: apply, cmp, execfile etc
  * using removed special methods: __coerce__, __delslice__ etc
  * using map / filter / reduce in non iterating context

-----

Pylint - Python 3 porting checker 
=================================

.. code-block:: sh

    def download_url(url):
        ...    
    map(download_url, urls) # download_url will never be called

    class A:
        __metaclass__ = type
        def __setslice__(self, other):
           if not isinstance(other, basestring):           
               ...

  $ pylint a.py --py3k

  W:  5, 0: map built-in referenced when not iterating
  W:  7, 0: Assigning to a class's __metaclass__ attribute
  W:  9, 8: __setslice__ method defined
  W: 10,36: basestring built-in referenced

----


Similar tools: pyflakes
=======================
 

* pyflakes: lightweight, fast, but detects only handful of errors

* promises not to have false positives or to warn about
  style issues

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

* still detects issues that most of static analyzers don't detect

   .. code-block:: python

      $ pychecker a.py
   
      a.py:2: Unpacking 3 values into 2 variables
      a.py:4: Using a conditional statement with a constant value
      a.py:6: Catching a non-Exception object (True)

-------


Similar tools: jedi and mypy
============================

* jedi: autocompletion library, wants to be a static analyzer, a lot of hardcoded behaviour

   .. code-block:: python

       $ python -m jedi linter a.py
       $ # it detected nothing :(

* mypy: optional type checker, with support for type hints through annotations,
  Guido loves it, PEP 484 started from here. Still work in progress.

   .. code-block:: python

     $ mypy a.py
     a.py: In function "test":
     a.py,line 2: Too many values to unpack (2 expected, 3 provided)
   
------

Static analysis shortcomings
============================

* static analysis is great

* but you can't fully understand code when:

   * dynamic code is invoked

   * extension modules are involved

   * you don't understand flow control

   * the code you're supposed to understand is too **smart** (namedtuple, enum, six.moves)

--------------

Static analysis shortcomings
============================

* Some users actually expect static analysis tools to understand this kind of code

  * nose.trivial

     .. code-block:: python

        for at in [ at for at in dir(_t)
                   if at.startswith('assert') and not '_' in at ]:
          pepd = pep8(at)
          vars()[pepd] = getattr(_t, at)
          __all__.append(pepd)

  * multiprocessing

     .. code-block:: python

        globals().update(
           (name, getattr(context._default_context, name))
           for name in context._default_context.__all__)
   
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

But, but.. how do I stop worrying and start loving the bugs?
============================================================

* write as many tests as you can, there is no such thing as **too many tests**

* use static analysis tools, any tool is better than nothing

* hopefully, you're going to use Pylint ;-)


--------


.. class:: center

    .. class:: title

    **Thank you!**

