# progrium/bashstyle (中文翻译 yujiaao)

Bash 是系统编程的 JavaScript. 但有时使用系统编程语言如 C 或 Go 会更好, Bash 是理想的面向 POSIX 标准的小任务或命令行级系统编程语言, 下面是三个显见的原因:

 * 它无处不在. 尤如 JavaScript 对 web 而言, Bash 在这里并为系统编程已准备好.
 * 它是中性的. 不同于 Ruby, Python, JavaScript, 或 PHP, Bash 平等地得罪了全部社群. ;)
 * 它生为胶水. 复杂部分用 C 或 Go (或其他什么语言!)来写, 然后用 Bash 把他们粘在一块.

本文档描述了我怎么写 Bash 以及我希望如何与合作者一起在我的开源项目里写 Bash. 本文基于大量经验和花了大量时间收集最佳实践. 多数来自这 
[两篇](http://wiki.bash-hackers.org/scripting/obsolete) 
[文章](http://www.kfirlavi.com/blog/2012/11/14/defensive-bash-programming/), 
但此处集成, 略有修改, 并聚焦在最值钱的点上. 加上一些新东西!

记住本文不是讲一般的 shell 脚本编程, 这里是专门针对 Bash 和 Bash 作为解释器的规则. 

## 大规则

 * 总是用双引号括起变量, 包括子shell. 不要裸体的 `$` 符号.
   * 这规则会带你走老远了. 详见 http://mywiki.wooledge.org/Quotes
 * 全部代码进函数. 即使只有一个函数, `main`. 
   * 除非是库脚本, 你可以全局进行脚本设置并调用 `main`. 如此而已.
   * 避免使用全局变量. 如定义常量则使用 `readonly`
 * 可执行脚本总有 `main` 函数, 调用时用 `main` 或 `main "$@"`
   * 如果脚本也用作库, 调用时用 `[[ "$0" == "$BASH_SOURCE" ]] && main "$@"` 判断
 * 设置变量时总是用 `local` 修饰, 除非有理由用 `declare`
   * 少数例外情况是当你故意想在外层空间设置一个变量.
 * 变量都用小写, 除非想输出到环境中.
 * 总用 `set -eo pipefail`. 快速失败并检查退出状态(exit codes). 
   * 在那些你有意让退出状态不为零的程序里使用 `|| true` .
 * 切勿使用弃用的样式. 最值得注意的:
   * 函数定义成 `myfunc() { ... }`, 而不是 `function myfunc { ... }`
   * 判断条件总是用 `[[` 替代 `[` 或 `test`
   * 从不使用反引号, 使用 `$( ... )`
   * 更多详见 http://wiki.bash-hackers.org/scripting/obsolete 
 * 优先使用绝对路径 (借助 $PWD), 总用 `./` 来修饰相对路径.
 * 两行以上的函数总是在顶部用 `declare` 声名变量和为参数命名
   * 如: `declare arg1="$1" arg2="$2"`
   * 定义可变参数函数时例外。见下文.
 * 使用 `mktemp` 创建临时文件, 总是用 `trap` 来清理.
 * 警告和错误应该发送到 STDERR, 任何可解析的内容都应该转到 STDOUT.
 * 完成后尝试本地化 `shopt` 使用和禁用选项.

如果你知道你在做什么，你可以曲解或破坏其中的一些规则，但通常它们是正确的并且非常有帮助。


## 最佳实践和技巧

 * 如果可能，请在 awk/sed 之前使用 Bash 变量替换.
 * 通常使用双引号，除非使用单引号更有意义.
 * 对于简单的条件，请尝试使用 `&&` 和 `||`.
 * 不要害怕 `printf`, 它比 `echo` 更强大.
 * 把 `then`, `do`, 等入在同一行, 不换行.
 * 如果您可以测试退出代码, 请避免在 if 表达式中中使用 `[[ ... ]]`.
 * 如果要包含/源码执行文件请使用 `.sh` 或 `.bash` 扩展名. 但不用在可执行脚本上.
 * 把复杂的单行 `sed`, `perl`命令等放在一个独立的函数中, 且使用描述性名称.
 * 包含 `[[ "$TRACE" ]] && set -x` 是个好主意.
 * 做简单明了的设计.
   * 避免使用选项标志和解析，而是尝试使用可选的环境变量.
   * 将子命令用于必要的不同“模式”.
 * 在大型系统或任何CLI命令中，向函数添加描述.
   * 在函数顶部使用 `declare desc="description"`，甚至在参数声明之上.
   * 这可以使用反射查询/提取。例如:
   ```
   eval $(type FUNCTION_NAME | grep 'declare desc=') && echo "$desc"
   ```
 * 意识到需要可移植性。在容器中运行 Bash 可以比 Bash 在多个平台上运行做出更多的假设.
 * 在期望或导出环境变量时，考虑可能涉及子 shell 的命名空间变量. 
 * 使用硬tab符。Heredocs忽略了前导tab符，允许更好的缩进.
 
## 好的参考和帮助

 * http://wiki.bash-hackers.org/scripting/start
   * 特别是 http://wiki.bash-hackers.org/scripting/newbie_traps
 * http://tldp.org/LDP/abs/html/
 * 交互式Bash提示: http://samrowe.com/wordpress/advancing-in-the-bash-shell/
 * 供参考, [谷歌的Bash风格指南](https://google.github.io/styleguide/shell.xml)
 * 对于标准检核, [shellcheck](https://github.com/koalaman/shellcheck)

## 例子

### 具有命名参数的常规函数
使用参数定义函数
```bash
regular_func() {
	declare arg1="$1" arg2="$2" arg3="$3"

	# ...
}
```

### 变参函数
使用最终的可变参数定义函数

```bash
variadic_func() {
	local arg1="$1"; shift
	local arg2="$1"; shift
	local rest="$@"

	# ...
}
```

### 条件：测试退出代码与输出

```bash
# 测试退出代码（-q静音输出）
if grep -q 'foo' somefile; then
  ...
fi

# 测试输出（ -m1 限制一个结果）
if [[ "$(grep -m1 'foo' somefile)" ]]; then
  ...
fi
```

### 更多的待办事项

