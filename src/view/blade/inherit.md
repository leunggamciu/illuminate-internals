# 模板继承

Blade支持模板继承，在父模板中使用`@yield`指令来创建一个占位符，子模板通过`@extends`指令来继承父模板，然后就能在子模板中
使用`@section`指令来填充父模板中的占位符。

具体使用方法如下:

```php
//base.blade.php

@yield('title')

```

```php
//page.blade.php

@extends('base')

@section('title')
Title from page
@endsection
```

上述的模板文件经过Blade的编译以后，就变成了:

```php
//page.php

$__env->startSection('title');
Title from page
$__env->stopSection();

echo $__env->make('base', array_except(get_defined_vars(), array('__data', '__path')))->render();
```

先来看看`startSection`的实现:

```php
//src/Illuminate/View/Concerns/ManagesLayouts.php

/**
 * Start injecting content into a section.
 *
 * @param  string  $section
 * @param  string|null  $content
 * @return void
 */
public function startSection($section, $content = null)
{
    if ($content === null) {
        if (ob_start()) {
            $this->sectionStack[] = $section;
        }
    } else {
        $this->extendSection($section, $content instanceof View ? $content : e($content));
    }
}
```

`startSection`这里走的是if分支，逻辑非常简单，只是开启输出缓冲以及把section名称推入到`sectionStack`中。

接下来是`stopSection`的实现:

```php
//src/Illuminate/View/Concerns/ManagesLayouts.php


/**
 * Stop injecting content into a section.
 *
 * @param  bool  $overwrite
 * @return string
 * @throws \InvalidArgumentException
 */
public function stopSection($overwrite = false)
{
    if (empty($this->sectionStack)) {
        throw new InvalidArgumentException('Cannot end a section without first starting one.');
    }

    $last = array_pop($this->sectionStack);

    if ($overwrite) {
        $this->sections[$last] = ob_get_clean();
    } else {
        $this->extendSection($last, ob_get_clean());
    }

    return $last;
}
```

`stopSection`把在`startSection`中推入到`sectionStack`的section名称给弹出来，而且还通过`ob_get_clean`函数
把输出缓冲区中的内容拿出来，最后清除并关闭输出缓冲。由于`$overwrite`是假值，所以，还需要调用`extendSection`方法。


```php
//src/Illuminate/View/Concerns/ManagesLayouts.php

/**
 * Append content to a given section.
 *
 * @param  string  $section
 * @param  string  $content
 * @return void
 */
protected function extendSection($section, $content)
{
    if (isset($this->sections[$section])) {
        $content = str_replace(static::parentPlaceholder($section), $content, $this->sections[$section]);
    }

    $this->sections[$section] = $content;
}
```

`extendSection`接受section名称以及section输出缓冲区的内容，目前`$this->sections`还是一个空数组，所以这里只是简单地把section输出
缓冲区中的内容放到一个键值数组中，键为section名称，值为输出缓冲区的内容。

到目前为止，`@section`指令已经执行完毕了。接下来轮到`@extends`指令了，从编译后的文件来看，`@extends`的实质是把父模板渲染一遍。

而父模板只有一个`@yield`指令，而`@yield`指令最终是编译成:

```php
$__env->yieldContent('title');
```

也就是说，前面编译后的page文件实质上的内容应该是:

```php
//page.php

$__env->startSection('title');
Title from page
$__env->stopSection();

echo $__env->yieldContent('title');
```

接下来就是`yieldContent`的实现:


```php
//src/Illuminate/View/Concerns/ManagesLayouts.php

/**
 * Get the string contents of a section.
 *
 * @param  string  $section
 * @param  string  $default
 * @return string
 */
public function yieldContent($section, $default = '')
{
    $sectionContent = $default instanceof View ? $default : e($default);

    if (isset($this->sections[$section])) {
        $sectionContent = $this->sections[$section];
    }

    $sectionContent = str_replace('@@parent', '--parent--holder--', $sectionContent);

    return str_replace(
        '--parent--holder--', '@parent', str_replace(static::parentPlaceholder($section), '', $sectionContent)
    );
}
```

`yieldContent`主要的作用是按照给定的section名称，从`$this->sections`中拿到之前在`stopSection`方法中放进去的section缓冲区内容，
然后经过一些替换(目前并不起作用)，最后返回这些内容。实际上，这里进行的替换是为了能在`@section`和`@endsection`之间输出`@parent`这个
字面值，很明显，为了要输出`@parent`，我们需要在模板中写成`@@parent`。


除了使用`@yield`创建占位符以外，还能直接使用`@section`和`@show`指令：


```php
//base.blade.php

@section('sidebar')
Sidebar
@show

```

这样，直接在base页面就能显示这个section。如果要继承的话，也能在子页面中用`@section`和`@endsection`来覆盖父页面的section。

Blade把`@show`编译成`<?php echo $_env->yieldSection(); ?>`。`yieldSection`的实现也很简单：

```php
//src/Illuminate/View/Concerns/ManagesLayouts.php

/**
 * Stop injecting content into a section and return its contents.
 *
 * @return string
 */
public function yieldSection()
{
    if (empty($this->sectionStack)) {
        return '';
    }

    return $this->yieldContent($this->stopSection());
}
```

它只是结合了`stopSection`和`yieldContent`方法，前面已经分析过了。


我们还能在子模板中访问父模板section的内容，例如:

```php
//base.blade.php

@section('sidebar')
Sidebar
@show

```


```php
//page.blade.php

@section('sidebar')
    @parent
@endsection
```

直接在子模板中使用`@parent`指令，就能把父模板中的内容插入到子模板中。

当Blade遇到`@parent`指令时候，它会把它替换成一个特殊的标记符，格式如下:

```
'##parent-placeholder-' . sha1($section) . '##'
```

然后记录到`$__env`的一个静态属性:$parentPlaceholder中，它是一个键值数组，键为section的名称，而值就是那个
特殊的标记符。而具体的替换过程是在`extendSection`那个方法里，它使用了`str_replace`，找到子模板缓冲区的那个
特殊的标记，然后用父模板的内容替换了它，这样就能实现将父模板的内容插入到子模板中。


除了上述功能以外，还有一些在文档中没有提到的功能，一个是阻止section被继承，另外一个是连接子模板的section。

首先是阻止section被继承，它用到了`@overwrite`指令。假设有以下模板页:


```php
//base.blade.php

@yield('title')

```

```php
//page.blade.php

@extends('base')

@section('title')
Title from page
@overwrite
```

```php
//page2.blade.php

@extends('page')

@section('title')
Title from page2
@endsection
```

这样，当渲染page2页的时候，实际上它的section是没有起作用的，渲染出来的是来自page页面的section。而`@overwrite`只是编译成

```php
echo $__env->stopSection(true);
```

它传递了一个真值给`stopSection`，导致Blade在渲染父页面的时候把之前在子页面中获得的section替换成父页面的section，因此子页面
的section也就不起作用了。

如果把`@overwrite`改成`@append`指令，那么渲染出来的页面就是

```
Title from page2Title from page
```

也就是把父页面的section拼接到子页面section的后面。`@append`指令最终编译成`<?php echo $__env->appendSection(); ?>`。

以下是`appendSection`的实现:

```php
//src/Illuminate/View/Concerns/ManagesLayouts.php

/**
 * Stop injecting content into a section and append it.
 *
 * @return string
 * @throws \InvalidArgumentException
 */
public function appendSection()
{
    if (empty($this->sectionStack)) {
        throw new InvalidArgumentException('Cannot end a section without first starting one.');
    }

    $last = array_pop($this->sectionStack);

    if (isset($this->sections[$last])) {
        $this->sections[$last] .= ob_get_clean();
    } else {
        $this->sections[$last] = ob_get_clean();
    }

    return $last;
}
```

和`stopSection`很像，主要逻辑的意思就是当渲染一个父页面时，看看子页面有没有定义对应的section，如果有，那么就把子页面section
的内容和父页面section的内容拼接起来，如果没有，那么就把父页面的section内容放到`$this->sections`数组中，这个数组的含义在之前
已经分析过了。