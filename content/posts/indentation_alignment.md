---
title: "什么样的代码缩进与对齐风格最合适？"
tags:
- TechTips
date: 2022-10-26
---
# 区别 Indentation 和 Alignment
Indentation 缩进有逻辑上的层级嵌套关系；而 Alignment 对齐逻辑上属于同一层级，存在只是出于美观需要。在如下代码中用 `>>>>` 代表 Indentation，`....` 代表 Alignment。
```C
int main()
{
>>>>int a = 1 + 
>>>>........2 + 3;
>>>>return 0;
}
```
Tab 和 Space 都可用于 Indentation 和 Alignment，根据用法分为多种不同的方案。

# Indentation, Alignment 方案
各种语言有自己的推荐的代码风格。

| Indentation/size | Alignment/size | Example |
| - | - | - |
| Tab/8 | Tab/8 | [Linux C coding style](https://www.kernel.org/doc/Documentation/process/coding-style.rst) |
| Tab/8 | Space/4 | [OpenBSD C style](https://man.openbsd.org/style) |
| Space/4 | Space/4 | [PEP8 - Style Guide for Python Code](https://peps.python.org/pep-0008/) |
| Tab | Tab/Space | [Effective Go](https://go.dev/doc/effective_go) |

注意上表中的 size 长度都是单位长度，实践中可以根据缩进层级或对齐美观需要使用多个单位。

使用 Tab 和 Space 各有优劣:
- 使用 Tab 比使用 Space 的文件字节数[更少](https://www.reddit.com/r/C_Programming/comments/auv5mg/file_size_impact_of_tabs_vs_spaces_in_c_code/)。
- Tab 更灵活。Tab 长度是由用户定义的，你想要多长都可以直接修改编辑器的配置，而不需要修改源文件。这是一把双刃剑，如果多人在一个项目上协同工作，每个人的 tabsize 不同，就不能一致地遵循 80 字符限制。

# 最佳实践
1. 优先遵循官方的标准规范，如 Python、Golang、Makefile。
2. 简单起见，C、ASM 使用 Tab/8 Indentation + Tab/8 Alignment。
3. 依赖代码格式化工具。Golang 可以完全依赖 `gofmt`；C 可以借助 Linux kernel 中的 scripts/checkpatch.pl 检测代码风格，scripts/Lindent 修改代码风格（依赖 `indent`）。
