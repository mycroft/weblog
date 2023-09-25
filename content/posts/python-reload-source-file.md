---
title: "Python: Reload source files"
date: 2011-03-26
summary: "Reload Python scripts without restarting whole apps"
tags:
  - python
---

When I was writing [BUTT - an irc bot](https://github.com/mycroft/Butt), I wanted to create a mini plugin system, to add or modify code without restarting any process. Python allows this natively using the [imp module](http://docs.python.org/library/imp.html). In the following examples, I'll show how to use the *load_source* procedure to load and parse a new source file.

Here a very simple module we'll import during execution time:

```python
#!/usr/bin/env python

class TestModule:
    def __init__(self):
        pass

    def myfunction(self):
        print "Calling myfunction"
```

Let's load module:

```python
#!/usr/bin/env python

import os, shutil
import imp

def load_module(module_name):
    print "Loading module %s" % module_name

    modfile = os.path.join(os.getcwd(), module_name + '.py')

    module = imp.load_source(module_name, modfile)

    print module
    # Returns <module 'test' from '/home/mycroft/imp/test.pyc'>

    print dir(module)
    # Permet de lister les differents attributs de l'objet importé.
    # Returns ['TestModule', '__builtins__', '__doc__', '__file__', '__name__', '__package__']

    for objname in dir(module):
        # Je veux specifiquement les modules suffixés "Module":
        if objname.endswith('Module'):
            modclass = getattr(module, objname)
            print modclass
            # Returns test.TestModule
            print dir(modlass)
            # Returns ['__doc__', '__init__', '__module__', 'myfunction']
            return modclass

    return None
```

Just testing the whole thing, with reloading a module:

```python
if __name__ == '__main__':

    modclass = load_module('test')
    instance = modclass()
    instance.myfunction()

    # We move away test.py, and copy test_new.py to test.py.

    print

    shutil.move('test.py', 'test_old.py')
    shutil.move('test_new.py', 'test.py')

    modclass = load_module('test')
    instance = modclass()
    instance.myfunction()

    shutil.move('test.py', 'test_new.py')
    shutil.move('test_old.py', 'test.py')
```

And we'll get:

```sh
$ ./modules.py
Loading module test
<module 'test' from '/home/mycroft/imp/test.py'>
['TestModule', '__builtins__', '__doc__', '__file__', '__name__', '__package__']
test.TestModule
['__doc__', '__init__', '__module__', 'myfunction']
Calling myfunction

Loading module test
<module 'test' from '/home/mycroft/imp/test.py'>
['TestModule', '__builtins__', '__doc__', '__file__', '__name__', '__package__']
test.TestModule
['__doc__', '__init__', '__module__', 'myfunction']
Calling (new) myfunction
```
