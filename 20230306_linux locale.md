

# 什么是locale

locale 这个单词中文翻译成地区或者地域，其实这个单词包含的意义要宽泛很多。locale 是根据计算机用户所使用的语言，所在国家或者地区，以及当地的文化传统所定义的一个软件运行时的语言环境。

通常，Locale 由语言(Language), 地域 (Territory) 和字符集(Codeset) ,修正值(optional codeset)组成。

```c
language[_territory][.codeset][@optional codeset]
```

而这些信息的配置通常放在/usr/share/i18n/locales 目录下。让我们打开以下zh_CN 和 en_US 。 实际上我们平时使用 date 函数的时候，就是在这里面进行格式选择的。

对于de_DE@euro的一点说明，@后边是修正项，也就是说你可以看到两个德国的locale：/usr/share/i18n/locales/de_DE@euro和/usr/share/i18n/locales/de_DE。打开这两个locale定义，你就会知道它们的差别在于de_DE@euro使用的是欧洲的排序、比较和缩进习惯，而de_DE用的是德国的标准习惯。

# locale分类

locale 把按照所涉及到的使用习惯的各个方面分成12 大类，这12 个大类分别是：

| 类别              | 描述                                                             |
| ----------------- | ---------------------------------------------------------------- |
| LC_CTYPE          | 语言符号及其分类                                                 |
| LC_NUMBERIC       | 数字                                                             |
| LC_COLLATE        | 比较习惯                                                         |
| LC_TIME           | 时间显示格式                                                     |
| LC_MONETARY       | 货币单位                                                         |
| LC_MESSAGES       | 信息主要是提示信息，错误信息，状态信息，标题，标签，按钮和菜单等 |
| LC_NAME           | 行么书写方式                                                     |
| LC_ADDRESS        | 地址书写方式                                                     |
| LC_TELEPHONE      | 电话号码书写方式                                                 |
| LC_MEASUREMENT    | 度量衡表达方式                                                   |
| LC_PAPER          | 默认纸张尺寸大小                                                 |
| LC_IDENTIFICATION | 对locale 自身包含信息的概述                                      |

# LC_ALL、LANG和LC_* 的关系

 设定locale 就是设定12 大类的locale分类属性，即12个LC_*。除了这个12个变量可以设定以外，为了简便起见，还有两个变量：LC_ALL 和LANG。他们之间有一个优先级的关系：LC_ALL > LC_* > LANG 可以这么说，LC_ALL 是最上级设定或者强制设定，而LANG 是默认设定值。

 在设置了LC_ALL 之后，如果不 把LC_ALL 强制设置为空，那么LC_* 将无法被设置。但是LANG 的值不受影响。

 LC_ALL，LANG和LC_*均不指定特定值的话，系统将采用POSIX作为lcoale，也就是C locale。

# **另外LANG和LANGUAGE有什么区别呢？**

LANG - Specifies the default locale for all unset locale variables
LANGUAGE - Most programs use this for the language of its interface
LANGUAGE是设置应用程序的界面语言。而LANG是优先级很低的一个变量，它指定所有与locale有关的变量的默认值，

# LC_ALL=C 的含义

 LC_ALL=C 是为了去除所有本地化的设置，让命令能正确执行。

"C"是系统默认的locale。所以当我们新安装完一个系统时，默认的locale就是C或POSIX。

“POSIX”:指定的最小环境c语言翻译称为POSIX locale。

"POSIX"是"C"的别名,如果不调用setlocale (), POSIX locale是默认的“C”相当于“POSIX”。

# locale 可以设置类型

```shell
$ locale -a
C
C.UTF-8
en_HK.utf8
POSIX
```
