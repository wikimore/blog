---
layout: post
title: Mustache语法说明
date: 2016-06-06 15:10:40
tags: mustache
categories: 技术
---

Mustache是一个无逻辑(logic-less)的模板系统，可以生成HTML、源代码、配置文件等等，目前我们在代码生成上就使用它。

#### 语法

Mustache的语法中主要是一个一个的标签(tag)，不同的标签有不同的含义，基本以多个`{}`形式展现。

- `{% raw %}{{}}{% endraw %}`
- `{% raw %}{{{}}}{% endraw %}`
- `{% raw %}{{&}}{% endraw %}`
- `{% raw %}{{#}}{{/}}{% endraw %}`
- `{% raw %}{{^}}{{/}}{% endraw %}`
- `{% raw %}{{!}}{% endraw %}`
- `{% raw %}{{>}}{% endraw %}`
- `{% raw %}{{= =}}{% endraw %}`

##### Variable

`{% raw %}{{}}{% endraw %}`表示一个值，如`{% raw %}hello, {{name}}!{% endraw %}`，如果给定一个Hash`{"name":"world"}`，生成的目标为`hello, world!`。

`{% raw %}{{{}}}{% endraw %}`和`{% raw %}{{&}}{% endraw %}`同样可以表示一个值，不过内部的数据是为转码(unescape)的，如`{% raw %}hello, {{name}}!{% endraw %}`，如果给定一个Hash`{"name":"<b>world</b>"}`，生成的目标为`hello, <b>world</b>!`，而用`{% raw %}{{}}{% endraw %}`则生成`hello, &lt;b&gt;world&lt;/b&gt!`

##### Section

###### true or false场景

如果模板如下：

```
begin
{% raw %}{{#display}}{% endraw %}
this line is display
{% raw %}{{/display}}{% endraw %}
end
```

给定json

```
{"display":true}
```

显示如下：

```
begin
this line is display
end
```

如果给定json

```
{"display":false}
```

显示如下：

```
begin
end
```

###### 列表场景

模板如下：

```
{% raw %}{{#persons}}
hello {{name}}
{{/persons}}{% endraw %}
```

给定json

```
{
  "persons": [
    { "name": "peter" },
    { "name": "leo" },
    { "name": "tom" }
  ]
}
```

显示如下：

```
hello peter
hello leo
hello tom
```

如果当列表为空时显示其他内容呢？我们可以编写如下模板：

```
{% raw %}
{{#persons}}
  <b>{{name}}</b>
{{/persons}}
{{^persons}}
  No person.
{{/persons}}
{% endraw %}
```

给定json

```
{
  "persons": []
}
```

显示如下：

```
No person.
```

##### Comment

{% raw %}{{! comment}}{% endraw %}，用!表示注释

```
{% raw %}<h1>Hello{{! comment }}.</h1>{% endraw %}
```

渲染如下：

```
<h1>Hello.</h1>
```

##### Partial

如果模板比较复杂，Mustache支持多个模板文件嵌套，使用语法 {% raw %}{{> sub}}{% endraw %}

```
{% raw %}
interface.mustache:
public interface {{name}} {
{{#methods}}
  {{> method}}
{{/methods}}
}
{% endraw %}
method.mustache:
public void {{name}}();
```

给定JSON

```
{
    "name": Iface
    "methods":[
        "name": "m1",
        "name": "m2",
        "name": "m3",
    ]
}
```

生成如下：

```
public interface Iface {
    public void m1();
    public void m2();
    public void m3();
}
```

##### Delimiter

Mustache使用`{% raw %}{{}}{% endraw %}`作为定界符，如果我们需要显示{}呢？貌似不可以。

其实Mustache支持设置定界符，使用等号，如`{% raw %}{{=<% %>=}}{% endraw %}`，这时候定界符就被替换成`<% %>`了，如果想重新设置回去，使用`{% raw %}<%={{ }}= %>{% endraw %}`。


目前几乎所有的主流语言都有对应的库，详见[Mustache](http://mustache.github.io/)。

