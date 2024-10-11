---
title: "Python Modules 机制简述"
tags:
- Python
date: 2024-09-16
---
# Module 作为脚本执行
当以 `python module.py` 运行 module 时，`__name__` 会被设置为 `__main__`，所以在模块尾部添加以下代码可以使该代码**仅在 module 作为主文件执行时运行**：
```python
if __name__ == "__main__":
    pass
```
# Module 搜索路径
当导入 spam 这个 module 时，解释器会
1. 先在 `sys.builtin_module_names` 中搜索，看是不是 built-in modules
2. 如果没有找到，就会在 `sys.path` 中搜索 spam.py 文件

`sys.path` 的值从以下位置初始化：
- 第一个位置：当前脚本所在的目录；若没有指定脚本（REPL），则就是一个代表当前工作目录的空字符串
- `PYTHONPATH`
- 依赖于安装的默认值（按惯例包含 `site-packages` 目录）

# \_\_pycache\_\_
为了加快 module 的**加载速度**，Python 将每个 module 的编译版本保存为 \_\_pycache\_\_ 目录下的 module.version.pyc 文件。从 .pyc 文件读取程序的运行速度并不比从 .py 文件读取程序的速度快； .pyc 文件唯一更快的是它们的加载速度。

# Packages
**注意**：所有 packages 都是 modules，但反之不然。换句话说，**Package 是一种特殊的 module**。特别地，任何包含 `__path__` 的 module 被视为一个 pacakge。为便于理解，可以把 module 视为 .py 文件，而 packages 视为包含 subpackages 和 submodules 的目录 (**but don’t take this analogy too literally**)。

Python 有两种类型的 package：regular package 和 namespace package。一个 regular package 目录下必须包含 \_\_init\_\_.py 文件，导入包时该包及其父包的 \_\_init\_\_.py 会被隐式执行。\_\_init\_\_.py 可以只是一个空文件，也可以执行包的初始化代码或设置 `__all__` 变量：
- 如果 \_\_init\_\_.py 中定义了一个名为 `__all__` 的列表，则当遇到 `from package import *` 时，它将被视为应导入的 modules 列表。
- 如果未定义 `__all__`，则 `from package import *` 语句不会导入所有 submodules，只确保 package 被导入（可能运行 \_\_init\_\_.py 中的任何初始化代码），然后导入包中定义的任何名称。

以下为一个可能的包结构：
```
sound/                          Top-level package
      __init__.py               Initialize the sound package
      formats/                  Subpackage for file format conversions
              __init__.py
              wavread.py
              wavwrite.py
              aiffread.py
              aiffwrite.py
              auread.py
              auwrite.py
              ...
      effects/                  Subpackage for sound effects
              __init__.py
              echo.py
              surround.py
              reverse.py
              ...
      filters/                  Subpackage for filters
              __init__.py
              equalizer.py
              vocoder.py
              karaoke.py
              ...
```

- 当使用 `from package import item` 时，该 item 可以是 package 的 submodule 或 subpackage，也可以是 package 中定义的其他名称，如函数、类或变量。如：
    - 导入 package 中的 modules：`from sound.effects import echo`
    - 直接导入函数或变量：`from sound.effects.echo import echofilter`

- 当使用像 `import item.subitem.subsubitem` 这样的语法时，除最后一个外，每项都必须是一个 package；最后一项可以是 module 或 package，但不能是前一项中定义的类、函数或变量。如：`import sound.effects.echo`

### 包内引用
- **绝对导入**：Subpackage 可以使用绝对导入来引用同级包的 submodule。

例如，若 sound.filters.vocoder 需要使用 sound.effects 中的 echo，则可以使用 `from sound.effects import echo`。

- **相对导入**：还可以使用 `from module import name` 形式编写相对导入。这些导入使用 leading dots 来指示相对导入中涉及的当前包和父包。单个 dot 表示相对于当前包，之后每多一个 dot 表示上一层级。

例如，在 surround 模块中可以使用：

```python
from .echo import echofilter
from . import echo
from .. import formats
from ..filters import equalizer
```

**注意**：
- 绝对导入可以使用 `import <>` 或 `from <> import <>` 语法，但相对导入只能用 `from <> import <>`。
- 相对导入基于当前模块的名称 `__name__`。由于主模块的名称始终为 `__main__`，因此 Python 应用程序主模块必须始终使用绝对导入。

# References
- [Python Doc - Tutorial - Modules](https://docs.python.org/3/tutorial/modules.html)
- [Python Doc - Reference - Import - Packages](https://docs.python.org/3/reference/import.html#packages)
