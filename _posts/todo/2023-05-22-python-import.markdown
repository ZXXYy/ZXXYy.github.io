---
title:      "Python import"
date:       2023-05-22 14:00:00
author:     zxy
categories: ["Coding", "Python"]
tags: ["Python"]
mathjax: true
post: false
---



In practice, a module usually corresponds to one `.py` file containing Python code.

```python
# Note that this places pi in the global namespace and not within a math namespace.
from math import pi
print(pi)

# math acts as a namespace that keeps all the attributes of the module together. 
import math
print(math.pi)
```



a package is a Python module with an `__path__` attribute.

To create a Python package yourself, you create a directory and a [file named `__init__.py`](https://docs.python.org/reference/import.html#regular-packages) inside it. The `__init__.py` file contains the contents of the package when it’s treated as a module. 

 Directories without an `__init__.py` file are still treated as packages by Python. However, these won’t be regular packages, but something called **namespace packages**.

## Python’s Import Path

You can inspect Python’s import path by printing `sys.path`

1. The directory of the current script (or the current directory if there’s no script, such as when Python is running interactively)
2. The contents of the `PYTHONPATH` environment variable
3. Other, installation-dependent directories

https://realpython.com/python-import/#the-python-import-system
