---
layout: post
title: Python字符编码
category: lang
tags: [encoding, python]
---

最近要用python写了小工具将mysql的记录导出到文本，发觉经常报UnicodeDecodeError,就花时间简单的研究了下字符编码以及python对其的处理。

* 
{:toc}

字符编码
----------

数据存储在计算机系统以及信息交互为0/1编码，必须通过某种编码方式将文字、符号转换为数字，然后通过特定的实现将其存储到计算机系统里。

*ASCII* 

ASCII字符集为美国信息交互标准编码，通过7位编码拉丁字母，包含128个字符，无法显示重音字母。其国际标准为ISO 646。Ascii字符表可以通过man ascii显示

1. 0-31以及127为控制字符以及通讯专用字符，为不可打印字符
2. 32-126为字符，32为空格，48～57为0到9十个阿拉伯数字，65～90为26个大写英文字母，97～122号为26个小写英文字母，其余为一些标点符号、运算符号等。在c语言中通过isspace等相关的函数可以对字符进行判断

*Latin-1* 

由于ASCII只使用了7位编码，Latin-1为8位编码，其中前128个编码与ASCII一致。其国际标准为ISO 8859-1。Latin-1字符表可以通过man ISO_8859-1显示,
其中A0-FF为西欧字符，80-9F为控制字符

*Unicode*

<http://en.wikipedia.org/wiki/Unicode>

Unicode基于通用字符集UCS（ISO 10646），其包含所有的其他字符集标准，而且保证与其它字符集的双向兼容。

其定义了1,114,112个码位，区间为0-0x10FFFF。UCS标准定义了一个31位的字符集架构，首先根据最高字节将其分了128（2^7）个24位的组，然后每个组根据次高字节分为256（2^8）个面。每个平面根据第3个字节分为256行 （row），每行有256个码位（cell）。

group 0的平面0被称作BMP（Basic Multilingual Plane），通过2个字节表示65536个字符，也称为UCS-2。
其中0x0000-0x00FF与Latin-1编码一致

*UTF*

Unicode只是一种编码方式，具体的实现方式称为Unicode Transformation Format。

* UTF-8

UTF-8以字节为单位对Unicode进行编码。从Unicode到UTF-8的编码方式如下：
 
    Unicode编码(16进制)      UTF-8 字节流(二进制)
    000000 - 00007F            0xxxxxxx
    000080 - 0007FF            110xxxxx 10xxxxxx
    000800 - 00FFFF            1110xxxx 10xxxxxx 10xxxxxx
    010000 - 10FFFF            11110xxx 10xxxxxx 10xxxxxx 10xxxxxx

UTF-8的特点是对不同范围的字符使用不同长度的编码。对于0x00-0x7F之间的字符，UTF-8编码与ASCII编码完全相同。UTF-8编码的最大长度是4个字节。

* UTF-16

UTF-16编码以16位无符号整数为单位，假设其字符编码为U，则编码方式为

    U<0x10000    U
    U≥0x10000    U`=U-0x10000，转为20位的二进制，其中前10位加前缀组成110110，后10为加前缀110111

* UTF-32

UTF-32编码使用32位进行编码，无需进行转换

*字节序* 

Unicode标准建议用BOM（Byte Order Mark）来区分字节序，即在传输字节流前，先传输被作为BOM的字符"零宽无中断空格"。这个字符的编码是FEFF。对于UTF-8，其使用EF BB BF来标识编码

java编译后的class文件通过UTF-16编码，加载到jvm中也是通过UTF-16编码。对于从数据库以及文件读取的字节流而言，可能为了存储优化，编码方式不同，需要根据源地址的编码进行转换

文件编码查看以及转换
----------------------

可以通过file <filename>查看文件的编码。
可以通过unicode命令查看字符的unicode编码以及通过unicode编码查看字符。
通过iconv进行文件编码的转换

例如：
    $ echo 中国 > uni.txt
    
    $ file uni.txt
         uni.txt: UTF-8 Unicode text
    
    $ hexdump uni.txt
         0000000 b8e4 e5ad bd9b 000a                    
         0000007
    
    $ unicode 中国
    U+4E2D CJK UNIFIED IDEOGRAPH-4E2D
    UTF-8: e4 b8 ad  UTF-16BE: 4e2d  Decimal: &#20013;
    中 (中)
    Uppercase: U+4E2D
    Category: Lo (Letter, Other)
    Bidi: L (Left-to-Right)
    
    U+56FD CJK UNIFIED IDEOGRAPH-56FD
    UTF-8: e5 9b bd  UTF-16BE: 56fd  Decimal: &#22269;
    国 (国)
    Uppercase: U+56FD
    Category: Lo (Letter, Other)
    Bidi: L (Left-to-Right)
    
    $iconv -f UTF-8 -t UTF-16 uni.txt > uni16.txt
    $hexdump -C uni16.txt 
    00000000  ff fe 2d 4e fd 56 0a 00                           |..-N.V..|
    00000008
    
    $file uni16.txt
    uni16.txt: Little-endian UTF-16 Unicode text, with no line terminators

通过vim打开uni16.txt，使用命令set fileencoding=utf-8，然后保存

    $file uni16.txt
    uni16.txt: UTF-8 Unicode (with BOM) text
    $ hexdump -C uni16.txt
    00000000  ef bb bf e4 b8 ad e5 9b  bd 0a                    |..........|
    0000000a

Python源码文件编码
--------------------

注释为 #,如果是中文注释，在第一行添加#coding=utf-8
或者通过
 # -*- coding: <encoding name> -*-

encoding注释必须在第一行或者第二行

python默认情况下认为源码的编码为latin-1

<http://www.python.org/dev/peps/pep-3120/>
<http://www.python.org/dev/peps/pep-0263/>

str以及unicode
---------------

str和unicode都是basestring的子类,str为通过utf-8编码后的字符序列,unicode是通过utf-16编码后的字符序列。

    >>> s='中国'
    >>> u=u'中国'
    >>> print repr(s)
    '\xe4\xb8\xad\xe5\x9b\xbd'
    >>> print repr(u)
    u'\u4e2d\u56fd'
    >>> print len(s)
    6
    >>> print len(u)
    2

string可以和unicode互相转换.默认编码可以sys.setdefaultencoding进行设置

    sd=s.decode('utf-8')
    >>> type(sd)
    <type 'unicode'>
    >>> print repr(sd)
    u'\u4e2d\u56fd'
    
    >>> us=unicode(s,encoding='utf-8')
    >>> print repr(us)
    u'\u4e2d\u56fd'
    
    >>> su=us.encode('utf-8')
    >>> print repr(su)
    '\xe4\xb8\xad\xe5\x9b\xbd'

unicode的方法实参可以为unicode类型或者str类型，不过str类型会通过默认的codec转换为unicode类型
通过unichr方法可以将10进制的码位与unicode进行转换，通过ord方法可以将unicode转为10进制的码位

对于文件读写，可以使用codecs提供的open方法打开，其会返回一个wrapped file对象，读取返回的为unicode对象。写入时，如果参数是unicode，则使用open()时指定的编码进行编码后写入；如果是str，则先根据源代码文件声明的字符编码，解码成unicode后再进行前述操作。
对于字符串format，如果format string为unicode则返回的也为unicode类型


参考资源
---------
<http://docs.python.org/2/howto/unicode.html>  
<http://blog.csdn.net/fmddlmyy/article/details/372148>  
<http://hektor.umcs.lublin.pl/~mikosmul/computing/articles/linux-unicode.html>

-EOF-

{% include references.md %}
