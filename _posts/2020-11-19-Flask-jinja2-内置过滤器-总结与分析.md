---
layout:     post
title:      Flask jinja2 内置过滤器 总结与分析
subtitle:   从源码上分析一下几个常用过滤器，以及不常见的用法，
date:       2020-11-19
author:     BY Diego
header-img: https://blog-1300884845.cos.ap-shanghai.myqcloud.com/background/jinja-filter.jpg
catalog: true
tags:
    - Python
---





[TOC]



# flask jinja2 内置过滤器



从jinja2 模板的源码看 `Lib\site-packages\jinja2\filters.py` 存在如下对应关系

```python
FILTERS = {
    'abs':                  abs,
    'attr':                 do_attr,
    'batch':                do_batch,
    'capitalize':           do_capitalize,
    'center':               do_center,
    'count':                len,
    'd':                    do_default,
    'default':              do_default,
    'dictsort':             do_dictsort,
    'e':                    escape,
    'escape':               escape,
    'filesizeformat':       do_filesizeformat,
    'first':                do_first,
    'float':                do_float,
    'forceescape':          do_forceescape,
    'format':               do_format,
    'groupby':              do_groupby,
    'indent':               do_indent,
    'int':                  do_int,
    'join':                 do_join,
    'last':                 do_last,
    'length':               len,
    'list':                 do_list,
    'lower':                do_lower,
    'map':                  do_map,
    'min':                  do_min,
    'max':                  do_max,
    'pprint':               do_pprint,
    'random':               do_random,
    'reject':               do_reject,
    'rejectattr':           do_rejectattr,
    'replace':              do_replace,
    'reverse':              do_reverse,
    'round':                do_round,
    'safe':                 do_mark_safe,
    'select':               do_select,
    'selectattr':           do_selectattr,
    'slice':                do_slice,
    'sort':                 do_sort,
    'string':               soft_unicode,
    'striptags':            do_striptags,
    'sum':                  do_sum,
    'title':                do_title,
    'trim':                 do_trim,
    'truncate':             do_truncate,
    'unique':               do_unique,
    'upper':                do_upper,
    'urlencode':            do_urlencode,
    'urlize':               do_urlize,
    'wordcount':            do_wordcount,
    'wordwrap':             do_wordwrap,
    'xmlattr':              do_xmlattr,
    'tojson':               do_tojson,
}
```



## abs、min、max、round过滤器 

直接映射 python内置的`abs`，

python手册：*返回一个数的绝对值。实参可以是整数或浮点数。如果实参是一个复数，返回它的模 。*

用法：`-1|abs`



min 用法：`[1,2,3,45,0]|min`

max 用法 `[1,2,3,45,0]|max`



`round`

官方解释 ：`Round the number to a given precision. The first parameter specifies the precision (default is ``0``)`

将数字四舍五入到给定的精度



源码

```python
def do_round(value, precision=0, method="common"):
    """Round the number to a given precision. The first
    parameter specifies the precision (default is ``0``), the
    second the rounding method:

    - ``'common'`` rounds either up or down
    - ``'ceil'`` always rounds up
    - ``'floor'`` always rounds down

    If you don't specify a method ``'common'`` is used.

    .. sourcecode:: jinja

        { { 42.55|round } }
            -> 43.0
        { { 42.55|round(1, 'floor') } }
            -> 42.5

    Note that even if rounded to 0 precision, a float is returned.  If
    you need a real integer, pipe it through `int`:

    .. sourcecode:: jinja

        { { 42.55|round|int } }
            -> 43
    """
    if method not in {"common", "ceil", "floor"}:
        raise FilterArgumentError("method must be common, ceil or floor")
    if method == "common":
        return round(value, precision)
    func = getattr(math, method)
    return func(value * (10 ** precision)) / (10 ** precision)
```



## length、count、wordcount过滤器

都直接映射 python内置的`len`

python手册: *返回对象的长度（元素个数）。实参可以是序列（如 string、bytes、tuple、list 或 range 等）或集合（如 dictionary、set 或 frozen set 等）。*

用法:`[1,2,3,4]|length`,`[1,2,3,4]|count`



`wordcount` 统计单词数

用法`'abcdef aa vv'|wordcount`  结果 3

## attr过滤器

对应`do_attr`，

官方解释：*Get an attribute of an object. `foo|attr("bar")` works like `foo["bar"]` just that always an attribute is returned and items are not looked up.*

即用来获取类的属性 由`getattr`函数来实现

源码如下

```python
@environmentfilter
def do_attr(environment, obj, name):
    """Get an attribute of an object.  ``foo|attr("bar")`` works like
    ``foo.bar`` just that always an attribute is returned and items are not
    looked up.

    See :ref:`Notes on subscriptions <notes-on-subscriptions>` for more details.
    """
    try:
        name = str(name)
    except UnicodeError:
        pass
    else:
        try:
            value = getattr(obj, name)
        except AttributeError:
            pass
        else:
            if environment.sandboxed and not \
               environment.is_safe_attribute(obj, name, value):
                return environment.unsafe_undefined(obj, name)
            return value
    return environment.undefined(obj=obj, name=name)
```



在jinja2中 ，获取属性有两种

```
{ { foo.bar } }
{ { foo['bar'] } }
```

差别

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201112102915.png)



这就是说这两种方式都会检查类中和类上（父类也可），但`attr过滤器`似乎没有能力去检查父类

如`ssti`中

`self._TemplateReference__context.request`  = `self._TemplateReference__context["request"]`

  ![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201109102319.png)

`self._TemplateReference__context|attr('request')`

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201109102454.png)

调试发现 `request类`在 parent 中，无法通过 `attr`中的`getattr` 获取

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201112103923.png)





## capitalize、lower、upper、title过滤器

`capitalize(s)`

对应 `do_capitalize`

官方解释：*Capitalize a value. The first character will be uppercase, all others lowercase.*

首字母大写 其他转小写

用法:`"abcdef"|capitalize` 

源码

```python
def do_capitalize(s):
    """Capitalize a value. The first character will be uppercase, all others
    lowercase.
    """
    return soft_unicode(s).capitalize()
```

lower、upper 同

源码

```python
def do_upper(s):
    """Convert a value to uppercase."""
    return soft_unicode(s).upper()


def do_lower(s):
    """Convert a value to lowercase."""
    return soft_unicode(s).lower()
```

`title` 

```python
def do_title(s):
    """Return a titlecased version of the value. I.e. words will start with
    uppercase letters, all remaining characters are lowercase.
    """
    return "".join(
        [
            item[0].upper() + item[1:].lower()
            for item in _word_beginning_split_re.split(soft_unicode(s))
            if item
        ]
    )
```



## center、truncate过滤器

`center(value, width=80)`

官方解释:*返回长度为 width 的字符串，原字符串在其正中。 使用指定的 fillchar 填充两边的空位（默认使用 ASCII 空格符）。 如果 width小于等于 len(s) 则返回原字符串的副本*

用法：`"abcdef"|center(30)`

`'            abcdef            '`

源码：

```python
	def do_center(value, width=80):
    """Centers the value in a field of a given width."""
    return text_type(value).center(width)
```





`truncate`

`返回字符串的截断副本。长度由第一个参数指定，默认值为255。如果第二个参数为true，则过滤器将按长度剪切文本。否则它将丢弃最后一个单词。如果文本实际上被截断，它将在后面附加省略号（“ ...”）。如果要使用与“ ...”不同的省略号，则可以使用第三个参数进行指定。`

源码

```python
@environmentfilter
def do_truncate(env, s, length=255, killwords=False, end="...", leeway=None):
    """Return a truncated copy of the string. The length is specified
    with the first parameter which defaults to ``255``. If the second
    parameter is ``true`` the filter will cut the text at length. Otherwise
    it will discard the last word. If the text was in fact
    truncated it will append an ellipsis sign (``"..."``). If you want a
    different ellipsis sign than ``"..."`` you can specify it using the
    third parameter. Strings that only exceed the length by the tolerance
    margin given in the fourth parameter will not be truncated.

    .. sourcecode:: jinja

        { { "foo bar baz qux"|truncate(9) } }
            -> "foo..."
        { { "foo bar baz qux"|truncate(9, True) } }
            -> "foo ba..."
        { { "foo bar baz qux"|truncate(11) } }
            -> "foo bar baz qux"
        { { "foo bar baz qux"|truncate(11, False, '...', 0) } }
            -> "foo bar..."

    The default leeway on newer Jinja versions is 5 and was 0 before but
    can be reconfigured globally.
    """
    if leeway is None:
        leeway = env.policies["truncate.leeway"]
    assert length >= len(end), "expected length >= %s, got %s" % (len(end), length)
    assert leeway >= 0, "expected leeway >= 0, got %s" % leeway
    if len(s) <= length + leeway:
        return s
    if killwords:
        return s[: length - len(end)] + end
    result = s[: length - len(end)].rsplit(" ", 1)[0]
    return result + end
```





## escape、e、forceescape过滤器



`escape = e`

官方解释：*Convert the characters &, <, >, ‘, and ” in string s to HTML-safe sequences. Use this if you need to display text that might contain such characters in HTML. Marks return value as markup string.*

将特殊字符转化为实体

用法：`[1,'<'',4]|escape` 保持原有类型

源码

```python
from markupsafe import escape # 引用外部库
```



`forceescape`

强制html转义

```
def do_forceescape(value):
    """Enforce HTML escaping.  This will probably double escape variables."""
    if hasattr(value, "__html__"):
        value = value.__html__()
    return escape(text_type(value))
```





## filesizeformat过滤器

`filesizeformat(value, binary=False)`

官方解释： *Format the value like a ‘human-readable’ file size (i.e. 13 kB, 4.1 MB, 102 Bytes, etc). Per default decimal prefixes are used (Mega, Giga, etc.), if the second parameter is set to True the binary prefixes are used (Mebi, Gibi).*

将数字转化为filesize格式

```python
def do_filesizeformat(value, binary=False):
    """Format the value like a 'human-readable' file size (i.e. 13 kB,
    4.1 MB, 102 Bytes, etc).  Per default decimal prefixes are used (Mega,
    Giga, etc.), if the second parameter is set to `True` the binary
    prefixes are used (Mebi, Gibi).
    """
    bytes = float(value)
    base = binary and 1024 or 1000
    prefixes = [
        (binary and "KiB" or "kB"),
        (binary and "MiB" or "MB"),
        (binary and "GiB" or "GB"),
        (binary and "TiB" or "TB"),
        (binary and "PiB" or "PB"),
        (binary and "EiB" or "EB"),
        (binary and "ZiB" or "ZB"),
        (binary and "YiB" or "YB"),
    ]
    if bytes == 1:
        return "1 Byte"
    elif bytes < base:
        return "%d Bytes" % bytes
    else:
        for i, prefix in enumerate(prefixes):
            unit = base ** (i + 2)
            if bytes < unit:
                return "%.1f %s" % ((base * bytes / unit), prefix)
        return "%.1f %s" % ((base * bytes / unit), prefix)
```



用法 ：`99999|filesizeformat`  结果：100.0 kB



## first 、last过滤器

`first(seq)`

官方解析：*Return the first item of a sequence.*

获取第一个元素

用法：`[1,2,3,4,5]|first`

源码

```python
@environmentfilter
def do_first(environment, seq):
    """Return the first item of a sequence."""
    try:
        return next(iter(seq))
    except StopIteration:
        return environment.undefined("No first item, sequence was empty.")
```

last 同 first

源码

```python
@environmentfilter
def do_last(environment, seq):
    """
    Return the last item of a sequence.

    Note: Does not work with generators. You may want to explicitly
    convert it to a list:

    .. sourcecode:: jinja

        { { data | selectattr('name', '==', 'Jinja') | list | last } }
    """
    try:
        return next(iter(reversed(seq)))
    except StopIteration:
        return environment.undefined("No last item, sequence was empty.")
```



## float、int、string、list、map过滤器

都是类型转化

直接贴源码

string 用的外部库

```python
"string": soft_unicode
```

其他

```python
def do_float(value, default=0.0):
    """Convert the value into a floating point number. If the
    conversion doesn't work it will return ``0.0``. You can
    override this default using the first parameter.
    """
    try:
        return float(value)
    except (TypeError, ValueError):
        return default

def do_int(value, default=0, base=10):
    """Convert the value into an integer. If the
    conversion doesn't work it will return ``0``. You can
    override this default using the first parameter. You
    can also override the default base (10) in the second
    parameter, which handles input with prefixes such as
    0b, 0o and 0x for bases 2, 8 and 16 respectively.
    The base is ignored for decimal numbers and non-string values.
    """
    try:
        if isinstance(value, string_types):
            return int(value, base)
        return int(value)
    except (TypeError, ValueError):
        # this quirk is necessary so that "42.23"|int gives 42.
        try:
            return int(float(value))
        except (TypeError, ValueError):
            return default

def do_list(value):
    """Convert the value into a list.  If it was a string the returned list
    will be a list of characters.
    """
    return list(value)

@contextfilter
def do_map(*args, **kwargs):
    """Applies a filter on a sequence of objects or looks up an attribute.
    This is useful when dealing with lists of objects but you are really
    only interested in a certain value of it.

    The basic usage is mapping on an attribute.  Imagine you have a list
    of users but you are only interested in a list of usernames:

    .. sourcecode:: jinja

        Users on this page: { { users|map(attribute='username')|join(', ') } }

    You can specify a ``default`` value to use if an object in the list
    does not have the given attribute.

    .. sourcecode:: jinja

        { { users|map(attribute="username", default="Anonymous")|join(", ") } }

    Alternatively you can let it invoke a filter by passing the name of the
    filter and the arguments afterwards.  A good example would be applying a
    text conversion filter on a sequence:

    .. sourcecode:: jinja

        Users on this page: { { titles|map('lower')|join(', ') } }

    Similar to a generator comprehension such as:

    .. code-block:: python

        (u.username for u in users)
        (u.username or "Anonymous" for u in users)
        (do_lower(x) for x in titles)

    .. versionchanged:: 2.11.0
        Added the ``default`` parameter.

    .. versionadded:: 2.7
    """
    seq, func = prepare_map(args, kwargs)
    if seq:
        for item in seq:
            yield func(item)
```



## format、replace、join过滤器

`format(value, *args, **kwargs)`

用法 同python format

三种使用方法

` "%s, %s!"|format(greeting, name) `

` "{}, {}!".format(greeting, name) `

` "%s, %s!" % (greeting, name) `

```python
def do_format(value, *args, **kwargs):
    if args and kwargs:
        raise FilterArgumentError(
            "can't handle positional and keyword arguments at the same time"
        )
    return soft_unicode(value) % (kwargs or args)
```



replace 同样的东西

```python
@evalcontextfilter
def do_replace(eval_ctx, s, old, new, count=None):
    """Return a copy of the value with all occurrences of a substring
    replaced with a new one. The first argument is the substring
    that should be replaced, the second is the replacement string.
    If the optional third argument ``count`` is given, only the first
    ``count`` occurrences are replaced:

    .. sourcecode:: jinja

        { { "Hello World"|replace("Hello", "Goodbye") } }
            -> Goodbye World

        { { "aaaaargh"|replace("a", "d'oh, ", 2) } }
            -> d'oh, d'oh, aaargh
    """
    if count is None:
        count = -1
    if not eval_ctx.autoescape:
        return text_type(s).replace(text_type(old), text_type(new), count)
    if (
        hasattr(old, "__html__")
        or hasattr(new, "__html__")
        and not hasattr(s, "__html__")
    ):
        s = escape(s)
    else:
        s = soft_unicode(s)
    return s.replace(soft_unicode(old), soft_unicode(new), count)
```



`join(value, d=u'', attribute=None)`

官方解释：*Return a string which is the concatenation of the strings in the sequence. The separator between elements is an empty string per default, you can define it with the optional parameter:*

同 python join函数

用法: `{"aa":1,"bb":2}|join()` # 结果 aabb

`[1,2,3]|join()` # 123



源码

```python
@evalcontextfilter
def do_join(eval_ctx, value, d=u"", attribute=None):
    """Return a string which is the concatenation of the strings in the
    sequence. The separator between elements is an empty string per
    default, you can define it with the optional parameter:

    .. sourcecode:: jinja

        { { [1, 2, 3]|join('|') } }
            -> 1|2|3

        { { [1, 2, 3]|join } }
            -> 123

    It is also possible to join certain attributes of an object:

    .. sourcecode:: jinja

        { { users|join(', ', attribute='username') } }

    .. versionadded:: 2.6
       The `attribute` parameter was added.
    """
    if attribute is not None:
        value = imap(make_attrgetter(eval_ctx.environment, attribute), value)

    # no automatic escaping?  joining is a lot easier then
    if not eval_ctx.autoescape:
        return text_type(d).join(imap(text_type, value))

    # if the delimiter doesn't have an html representation we check
    # if any of the items has.  If yes we do a coercion to Markup
    if not hasattr(d, "__html__"):
        value = list(value)
        do_escape = False
        for idx, item in enumerate(value):
            if hasattr(item, "__html__"):
                do_escape = True
            else:
                value[idx] = text_type(item)
        if do_escape:
            d = escape(d)
        else:
            d = text_type(d)
        return d.join(value)

    # no html involved, to normal joining
    return soft_unicode(d).join(imap(soft_unicode, value))
```





##  reverse、sort 、random、batch过滤器

`reverse`

官方解释 :*Reverse the object or return an iterator the iterates over it the other way round.*

没什么可解释的

```python
def do_reverse(value):
    """Reverse the object or return an iterator that iterates over it the other
    way round.
    """
    if isinstance(value, string_types):
        return value[::-1]
    try:
        return reversed(value)
    except TypeError:
        try:
            rv = list(value)
            rv.reverse()
            return rv
        except TypeError:
            raise FilterArgumentError("argument must be iterable")
```

`sort`

用法在下面注释里

源码

```python
@environmentfilter
def do_sort(environment, value, reverse=False, case_sensitive=False, attribute=None):
    """Sort an iterable using Python's :func:`sorted`.

    .. sourcecode:: jinja

        { % for city in cities|sort % }
            ...
        { % endfor % }

    :param reverse: Sort descending instead of ascending.
    :param case_sensitive: When sorting strings, sort upper and lower
        case separately.
    :param attribute: When sorting objects or dicts, an attribute or
        key to sort by. Can use dot notation like ``"address.city"``.
        Can be a list of attributes like ``"age,name"``.

    The sort is stable, it does not change the relative order of
    elements that compare equal. This makes it is possible to chain
    sorts on different attributes and ordering.

    .. sourcecode:: jinja

        { % for user in users|sort(attribute="name")
            |sort(reverse=true, attribute="age") % }
            ...
        { % endfor % }

    As a shortcut to chaining when the direction is the same for all
    attributes, pass a comma separate list of attributes.

    .. sourcecode:: jinja

        { % for user users|sort(attribute="age,name") % }
            ...
        { % endfor % }

    .. versionchanged:: 2.11.0
        The ``attribute`` parameter can be a comma separated list of
        attributes, e.g. ``"age,name"``.

    .. versionchanged:: 2.6
       The ``attribute`` parameter was added.
    """
    key_func = make_multi_attrgetter(
        environment, attribute, postprocess=ignore_case if not case_sensitive else None
    )
    return sorted(value, key=key_func, reverse=reverse)
```



`random(seq)`

从列表中随机选择一个元素

使用

`[1,2,3,4,5]|random`

源码

```python
@contextfilter
def do_random(context, seq):
    """Return a random item from the sequence."""
    try:
        return random.choice(seq)
    except IndexError:
        return context.environment.undefined("No random item, sequence was empty.")
```



`batch(value, linecount, fill_with=None)`

对应`do_batch`

官方解释：*A filter that batches items. It works pretty much like slice just the other way round. It returns a list of lists with the given number of items. If you provide a second parameter this is used to fill up missing items.*

简单来说就是将列表分割 , 每个有linecount个元素，不足的用file_with进行填充

用法：`{ % for row in ["1","22","333","4444"]|batch(3, '**') % }{ { row } }{ % endfor % }`

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201112112854.png)

源码:

```python
def do_batch(value, linecount, fill_with=None):
    """
    A filter that batches items. It works pretty much like `slice`
    just the other way round. It returns a list of lists with the
    given number of items. If you provide a second parameter this
    is used to fill up missing items. See this example:

    .. sourcecode:: html+jinja

        <table>
        { %- for row in items|batch(3, '&nbsp;') % }
          <tr>
          { %- for column in row % }
            <td>{ { column } }</td>
          { %- endfor % }
          </tr>
        { %- endfor % }
        </table>
    """
    tmp = []
    for item in value:
        if len(tmp) == linecount:
            yield tmp
            tmp = []
        tmp.append(item)
    if tmp:
        if fill_with is not None and len(tmp) < linecount:
            tmp += [fill_with] * (linecount - len(tmp))
        yield tmp
```





## safe过滤器

官方解释:`"Mark the value as safe which means that in an environment with automatic
    escaping enabled this variable will not be escaped`

将值标记为安全，这意味着在启用了自动转义的环境中，此变量将不会被转义。

源码

```python
def do_mark_safe(value):
    """Mark the value as safe which means that in an environment with automatic
    escaping enabled this variable will not be escaped.
    """
    return Markup(value)
```



## reject 、rejectattr、select、selectattr过滤器

`reject`

仅对未通过给定测试的项目进行迭代,通过对每个项目应用测试并在测试成功的地方拒绝这些项目来过滤Iterable。如果未指定测试，则将每个对象评估为布尔值

使用：`[1,3,4,9,16,48]|reject('>',3)|list` 结果 `[1, 3]`

第一个参数可选如下

![](https://blog-1300884845.cos.ap-shanghai.myqcloud.com/wenzhang/20201116093932.png)



`rejectattr`

用法 `({'a': 0}, {'a': 1}, {'a': 2})|rejectattr('a', '<', 2) | list` 结果`[{'a': 2}]` 参数二 同上图



`select`、`selectattr` 用法同上



实现代码都如下

```python
@contextfilter
def do_reject(*args, **kwargs):
    return select_or_reject(args, kwargs, lambda x: not x, False)
```



## trim 、striptags、urlencode、xmlattr、tojson过滤器

`trim` 去除两边的特定字符

用法`'bbaavvvaa'|trim('a')` 结果 `bbaavv`

```python
def do_trim(value, chars=None):
    """Strip leading and trailing characters, by default whitespace."""
    return soft_unicode(value).strip(chars)
```



`striptags` 去除标签

用法 `'222<11>333'|striptags`  结果`222333`



`urlencode` 编码

```python
def do_urlencode(value):
    """Quote data for use in a URL path or query using UTF-8.

    Basic wrapper around :func:`urllib.parse.quote` when given a
    string, or :func:`urllib.parse.urlencode` for a dict or iterable.

    :param value: Data to quote. A string will be quoted directly. A
        dict or iterable of ``(key, value)`` pairs will be joined as a
        query string.

    When given a string, "/" is not quoted. HTTP servers treat "/" and
    "%2F" equivalently in paths. If you need quoted slashes, use the
    ``|replace("/", "%2F")`` filter.

    .. versionadded:: 2.7
    """
    if isinstance(value, string_types) or not isinstance(value, abc.Iterable):
        return unicode_urlencode(value)

    if isinstance(value, dict):
        items = iteritems(value)
    else:
        items = iter(value)

    return u"&".join(
        "%s=%s" % (unicode_urlencode(k, for_qs=True), unicode_urlencode(v, for_qs=True))
        for k, v in items
    )

```



`xmlattr`

用法`{'class': 'my_list', 'missing': none, 'id': 'list'}|xmlattr`  结果`class="my_list" id="list"`

```python
@evalcontextfilter
def do_xmlattr(_eval_ctx, d, autospace=True):
    """Create an SGML/XML attribute string based on the items in a dict.
    All values that are neither `none` nor `undefined` are automatically
    escaped:

    .. sourcecode:: html+jinja

        <ul{ { {'class': 'my_list', 'missing': none,
                'id': 'list-%d'|format(variable)}|xmlattr } }>
        ...
        </ul>

    Results in something like this:

    .. sourcecode:: html

        <ul class="my_list" id="list-42">
        ...
        </ul>

    As you can see it automatically prepends a space in front of the item
    if the filter returned something unless the second parameter is false.
    """
    rv = u" ".join(
        u'%s="%s"' % (escape(key), escape(value))
        for key, value in iteritems(d)
        if value is not None and not isinstance(value, Undefined)
    )
    if autospace and rv:
        rv = u" " + rv
    if _eval_ctx.autoescape:
        rv = Markup(rv)
    return rv
```



# 参考

[模板设计者文档](http://docs.jinkan.org/docs/jinja2/templates.html#id1)



[Python 标准库](https://doc.bccnsoft.com/docs/python-3.7.3-docs-html-cn/library/#the-python-standard-library)



[Jinja Filters](https://docs.exponea.com/docs/filters)