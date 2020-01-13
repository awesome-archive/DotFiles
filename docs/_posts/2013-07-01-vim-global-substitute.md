---
layout: post
title:  "强大的vim命令 :g 和 :s"
categories: 学习笔记
tags: vim
author: 王世东
---

* content
{:toc}

`:global` 和 `:s` 命令是 Vim 最强大的命令之一，将其摸透用熟可以事半功倍，在这里我总结了一些网上的网上的经典问题，结合自己的使用和理解，通过实例详细介绍一下其用法。示例难度不一，其中有些可能并没有多少实用性，仅仅是为了展示功能。

## 命令形式

阅读 `:help :g`可知，该命令的使用形式为：

```
:[range]global/{pattern}/{command}
```
`global` 命令在 `[range]` 指定的文本范围内（缺省为整个文件）查找 `{pattern}`，然后对匹配到的行执行命令`{command}`，如果希望对没匹配上的行执行命令，则使用 `global!` 或 `vglobal` 命令。

先来看 Vim 用户手册里的一个经典例子。

【例1】倒序文件行（即unix下的tac命令）

```
:g/^/m 0
```

这条命令用行首标记 `^` 匹配文件的所有行（这是查找的一个常用技巧，如果用 `.`则是匹配非空行，不满足本例要求），然后用 `move` 命令依次将每行移到第一行（第0行的下一行），从而实现了倒序功能。

`global`命令实际上是分成两步执行：首先扫描`[range]`指定范围内的所有行，给匹配`{pattern}`的行打上标记；然后依次对打有标记的行执行`{command}`命令，如果被标记的行在对之前匹配行的命令操作中被删除、移动或合并，则其标记自动消失，而不对该行执行`{command}`命令。标记的概念很重要，以例说明。

【例2】删除偶数行

```
:g/^/+1 d
```

这条命令也是匹配所有行，然后隔行删除（其中+1用以定位于当前行的下一行）。为什么是隔行呢？因为在对第一行执行+1 d命令时删除的是第二行，而第二行虽然也被标记了，但已不存在了，因此不会执行删除第三行的命令。

本例也可以用`:normal`命令实现：

```
:%normal! jdd
```

`%`指定整个文件，然后依次执行普通模式下的`jdd`，即下移删除一行。与`global`命令不同之处在于，`%normal! jdd`是按照行号顺序执行，在第一行时删除了第二行，后面的所有行号都减一，因此在第二行执行jdd时删除的是原来的第四行。也就是说，global命令是通过偶数行标记的消失实现的，而normal命令是通过后续行的自动前移实现的。

【例3】删除奇数行

```
:g/^/d|m.
```
光是`:g/^/d`显然不行，这会删除所有行，我们需要用`move`命令把偶数行的标记去掉。当然，本例可以很简单的转换成【例2】，在此只是用来强调标记的概念。

本例若想用`normal`命令实现比较有意思，`%norm dd`也同样会删除整个文件，`%norm jkdd`就可以，我不知道两者为什么不同，可能和normal命令内部的运行机制有关。

## global与substitute

不少人觉得这两个命令差不多，的确，它们的形式很相似，都是要进行查找匹配，只不过substitute执行的是替换而global执行的其它命令（当然，`substitute`缺省的`[range]`是当前行，这点也不同）。先看两个例子，体会一下`:s`和`:g`不同的思维方式。

【例4】double所有行

```
:%s/.*/&\r&/
:g/^/t.
```

`substitue`是查找任意行，然后替换为两行夹回车；`global`是将每一行复制（`:t`就是`:copy`）到自己下面，更加清晰明了。

【例5】把以回车排版、以空行分段的文本变成以回车分段的文本

很多txt格式的ebook，以及像vim help这样的文本，每行的字符数受限，段之间用空行分隔。若把它们拷贝到word里，那些硬回车和空行就比较讨厌了，虽然word里也有自动调整格式的功能，不过在Vim里搞定更是小菜一碟。先看看用替换如何实现。

```
:%s/\n\n\@!//
```
`\n\n\@!`是查找后面不跟回车的回车（关于`\@!`的用法请`:h /\@!`，在此不多说了），然后替换为空，也就是去掉用于排版的回车。

注：如果等宽段之间无空行，则合并命令变为`:%s/\n\%(\s\{2,}\|\n\)\@!//g`

`global`命令则完全是另一种思路。

```
:g/./,/^$/j
```
`/./`标记非空行，`/^$/`查找其后的空行，然后对二者之间的行进行合并操作。也许有人会问，段中的每一行会不会都执行了`j`命令？前面已经说过，在之前操作中消失掉的标记行不执行操作命令，在处理每段第一行时已经把段内的其余行都合并了，所以每段只会执行一次`j`命令。这条命令使用`global`标记做为`[range]`的起始行，这样的用法后面还会详述。

`global`经常与`substitute`组合使用，用前者定位满足一定条件的行，用后者在这些行中进行查找替换。如：

【例6】将aaa替换成bbb，除非该行中有ccc或者ddd

```
:v/ccc\|ddd/s/aaa/bbb/g
```

【例7】将aaa替换成bbb，条件是该行中有ccc但不能有ddd

如何写出一个匹配aaa并满足行内有ccc但不能有ddd的正则表达式？我不知道。即便能写出来，也必定极其复杂。用global命令则并不困难：

```
:g/ccc/if getline('.') !~ 'ddd' | s/aaa/bbb/g
```

该命令首先标记匹配`ccc`的行，然后执行`if`命令（`if`也是`ex`命令！），`getline`函数取得当前行，然后判断是否匹配`ddd`，如果不匹配（`!~`的求值为`true`）则执行替换。要掌握这样的用法需要对`ex`命令、Vim函数和表达式有一定了解才行，实际上，这条命令已经是一个快捷版的脚本了。可能有人会想，把`g`和`v`连起来用不就行了么，可惜`global`命令不支持（恐怕也没法支持）嵌套。

## global标志的[range]用法

```
:h range
```

在`global`命令第一步中所设的标记，可以被用来为`{command}`命令设定各种形式的`[range]`。在【例2】和【例5】中都已使用了这一技巧，灵活使用`[range]`，是一项重要的基本功。先看看【例2】和【例3】的一般化问题。

【例8】每n行中，删除前/后m行（例如，每10行删除前/后3行）

```
:g/^/,+2 d | ,+6 m -1
:g/^/,+6 m -1 | +1,+3 d
```

这两个命令还是利用move来清除保留行的标志，需要注意的是执行第二个命令时的当前行是第一个命令寻址并执行后的位置。再看两个更实用点的例子。

【例9】提取条件编译内容。例如，在一个多平台的C程序里有大量的条件编译代码：

```
#ifdef WIN32
XXX1
XXX2
#endif
...
#ifdef WIN32
XXX3
XXX4
#else
YYY1
YYY2
#endif
```

现在用global命令把Win32平台下代码提取出来，拷贝到文件末：

```
:g/#ifdef WIN32/+1,/#else\|#endif/-1 t $
```

`t`命令的`[range]`是由逗号分隔，起始行是`/#ifdef WIN32/`标记行的下一行，结束行是一个查找定位，是在起始行后面出现的`#endif`或`#else`的上一行，`t`将二者间的内容复制到末尾。

【例10】提取上述C程序中的非Win32平台的代码（YYY部分）

首先说明一下，这个例子比前例要复杂的多，主要涉及的是`[range]`的操作，已经和global命令没多少关系，大可不看。加到这的目的是把问题说完，供喜欢细抠的朋友参考。本例的复杂性在于：首先，不能简单的用`#else`和`#endif`定位，因为代码中可能有其它的条件编译，我们必须要将查找范围限定在`#ifdef WIN32`的block中；另外，在block中可能并没有`#else`部分，这会给定位带来很大麻烦。解决方法是：

```
:try | g/#ifdef WIN32//#else/+1, /#endif/-1 t $ | endtry
```

先不管try和endtry，只看中间的`global`部分：找到`WIN32`，再向后找到`#else`，将其下一行作为`[range]`的起始行，然后从当前的光标（`WIN32`所在行，而非刚找到的`#else`的下一行）向下找到`#endif`，将其上一行作为`[range]`的结束行，然后执行`t`命令。但对于没有`#else`的block，如第一段代码，`[range]`的起始行是`YYY1`，而结束行是`XXX2`（因为查找`#endif`时是从第一行开始的，而不是从`YYY1`开始），这是一个非法的`[range]`，会引起exception，如果不放在try里面global命令就会立刻停止。

与逗号(,)不同，如果`[range]`是用分号(;)分隔的，则会使得当前光标移至起始行，在查找#endif时是从#else的下一行开始，这样就产生非法[range]，用不着try，但带来的问题是：没有#else的block会错误的把后面block中的#else部分找出来。

## global与Vim脚本

:h script
:h expression

经常有人问：XxEditor有个什么功能，Vim支持么？很可能不支持，因为Vim不大会为特定用户群提供非一般化的功能，但很少有什么功能不能在Vim定制出来，如果是你常用，就加到你的vimrc或者plugin里。脚本就是定制Vim的一种利器。本文不讨论脚本的编写，而是介绍如何实用global实现类似脚本的功能，实际上，就是利用命令提供的机制，做一个简化的脚本。

【例11】计算文件中数字列之和（或其它运算）

```
:let i=0
:g/^/let i+=str2nr(getline('.'))
:echo i
```

首先定义变量i并清零，然后用str2nr函数把当前行转成数字累加到i中，注意Vim不支持浮点数。global在这里实际上是替代了脚本里的for循环。 

Vim中最常见的一个问题是如何产生一列递增数字，有很多解决办法，调用外部命令，录宏，用`substitute`命令，还有专门的插件，

```
i 1 <Esc> qa yyp $ <Ctrl>-a q 99@a
:%s/^/\=printf('%d', line('.'))
:g/^/s//\=line('.')/
:let i=1|g/^/s//\=i/|let i=i+1 
```

而用global命令，可以实现一些更高级的功能。见下例。 

【例12】给有效代码行添加标号

在_Data Structures and Algorithm Analysis in C_一书中，作者为了便于讨论，将代码中的有效行加上注释标号，例如：

```
unsigned int factorial( unsigned int n )
{
if( n <= 1 )
return 1;
return( n * factorial(n-1) );
}
```

为了在添加标号后能对齐，我们预先在每行代码前面插入足够多的空格（这当然很简单），然后用global命令自动添加标号：

```
:let i=1 | g/\a/s/ \{8}/\=printf("\/* - *\/",i)/ | let i+=1
```

其中变量i用来记录标号，g命令查找有字母的行，然后把前8个空格替换成注释标号，每行处理完成后标号加一。替换中用到了/\=，一个非常有用的功能。

## 小结

要用好global命令并非易事，命令中的每一部分都值得仔细研究：只有掌握了range原理，才能自如的在文件中定位；只有精通pattern，才能有效的匹配到想要找的行；只有熟悉ex命令，才能选用最合适的功能进行操作；只有对变量、表达式、函数等内容有一定了解，才更能让global命令实现脚本的功能。总之，global是一个非常好的框架，对Vim越是熟悉，就越能将其种种武器架设在其上使用，发挥更大的威力。

global当然并非万能，功能也有所欠缺，最主要的问题是只能用正则表达式来标志匹配行，如果能用任意表达式来标记（或者从另一个角度，如前mv版主runsnake所说，引入求值正则表达式），则可实现更加方便功能。比如前述的几个删除特定行的问题，将会有简单而统一的解决方法。上述例子如果用sed、awk等专门的文本处理工具，或者perl之类的script语言也非难事，有些实现起来会更加方便。本文提供的Vim解决方法未必简单，甚至可能是难于理解，目的在于介绍global的使用。对于那些不会或者不能使用其它工具的朋友，参考价值可能更大一些。其实Vim的功能实在很丰富，值得我们深入学习。打个不恰当的比方，少林七十二绝技固然高妙，会的越多自然功力越强，不过只要会上一门六脉神剑或小无相功，也足以独步江湖了。