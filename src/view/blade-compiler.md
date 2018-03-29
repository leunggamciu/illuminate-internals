# Blade模板编译器

## 模板继承

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

$_env->startSection('title');
Title from page
$_env->stopSection();

echo $_env->make('base', array_except(get_defined_vars(), array('__data', '__path')))->render();
```

先来看看`startSection`的实现:

```php
//src/View/Concerns/ManagesLayouts.php

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
//src/View/Concerns/ManagesLayouts.php


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
//src/View/Concerns/ManagesLayouts.php

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
$_env->yieldContent('title');
```

也就是说，前面编译后的page文件实质上的内容应该是:

```php
//page.php

$_env->startSection('title');
Title from page
$_env->stopSection();

echo $_env->yieldContent('title');
```

接下来就是`yieldContent`的实现:


```php
//src/View/Concerns/ManagesLayouts.php

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