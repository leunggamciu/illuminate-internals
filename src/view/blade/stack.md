# stack

在Blade中，我们可以使用`@stack`指令定义可以占位符，然后用`@push`和`@prepend`指令往占位符里添加内容。更具体的用法请参考文档。

`@stack`指令经过编译后，变成了

```php
<?php echo $__env->yieldPushContent($expression); ?>
```

下面是`yieldPushContent`的实现

```php
//src/Illuminate/View/Concerns/ManagesStacks.php


/**
 * Get the string contents of a push section.
 *
 * @param  string  $section
 * @param  string  $default
 * @return string
 */
public function yieldPushContent($section, $default = '')
{
    if (! isset($this->pushes[$section]) && ! isset($this->prepends[$section])) {
        return $default;
    }

    $output = '';

    if (isset($this->prepends[$section])) {
        $output .= implode(array_reverse($this->prepends[$section]));
    }

    if (isset($this->pushes[$section])) {
        $output .= implode($this->pushes[$section]);
    }

    return $output;
}
```

这里主要有两个数组，一个是`$this->prepends`，另外一个就是`$this->pushes`，`yieldPushContent`主要就是
把这两个数组的内容合并到一起，然后将合并后的结果返回。

`@push`与`@endpush`指令最终是编译成下面的PHP代码:

```php
<?php $__env->startPush('foo'); ?>
<?php $__env->stopPush(); ?>
```

对应的实现如下：

```php
//src/Illuminate/View/Concerns/ManagesStacks.php

/**
 * Start injecting content into a push section.
 *
 * @param  string  $section
 * @param  string  $content
 * @return void
 */
public function startPush($section, $content = '')
{
    if ($content === '') {
        if (ob_start()) {
            $this->pushStack[] = $section;
        }
    } else {
        $this->extendPush($section, $content);
    }
}

/**
 * Stop injecting content into a push section.
 *
 * @return string
 * @throws \InvalidArgumentException
 */
public function stopPush()
{
    if (empty($this->pushStack)) {
        throw new InvalidArgumentException('Cannot end a push stack without first starting one.');
    }

    return tap(array_pop($this->pushStack), function ($last) {
        $this->extendPush($last, ob_get_clean());
    });
}

/**
 * Append content to a given push section.
 *
 * @param  string  $section
 * @param  string  $content
 * @return void
 */
protected function extendPush($section, $content)
{
    if (! isset($this->pushes[$section])) {
        $this->pushes[$section] = [];
    }

    if (! isset($this->pushes[$section][$this->renderCount])) {
        $this->pushes[$section][$this->renderCount] = $content;
    } else {
        $this->pushes[$section][$this->renderCount] .= $content;
    }
}
```

`startPush`的主要作用是开启输出缓冲，然后把stack的名称存起来。`endPush`则负责把输出缓冲区的内容
拿出来，关闭缓冲区，并且连同之前存起来的stack名称一起，传递给`extendPush`方法。而`extendPush`
就负责将所有push到stack里面的内容放到`$this->pushes`数组里。这里要注意一下的是`$this->renderCount`，
它是一个计数器，Blade每开始渲染一个模板，这个计数器就加一，模板渲染结束，计数器就减一，可以用它来跟踪
当前渲染到的模板在整个模板树中的层级。而这个计数器应用到这里表明了同一个层级要`@push`的内容会连接在一起。

`@prepend`的实现和`@push`的实现十分相似。它被编译成

```php
<?php $__env->startPrepend('foo'); ?>
<?php $__env->stopPrepend(); ?>
```

```php
//src/Illuminate/View/Concerns/ManagesStacks.php

/**
 * Start prepending content into a push section.
 *
 * @param  string  $section
 * @param  string  $content
 * @return void
 */
public function startPrepend($section, $content = '')
{
    if ($content === '') {
        if (ob_start()) {
            $this->pushStack[] = $section;
        }
    } else {
        $this->extendPrepend($section, $content);
    }
}

/**
 * Stop prepending content into a push section.
 *
 * @return string
 * @throws \InvalidArgumentException
 */
public function stopPrepend()
{
    if (empty($this->pushStack)) {
        throw new InvalidArgumentException('Cannot end a prepend operation without first starting one.');
    }

    return tap(array_pop($this->pushStack), function ($last) {
        $this->extendPrepend($last, ob_get_clean());
    });
}

/**
 * Prepend content to a given stack.
 *
 * @param  string  $section
 * @param  string  $content
 * @return void
 */
protected function extendPrepend($section, $content)
{
    if (! isset($this->prepends[$section])) {
        $this->prepends[$section] = [];
    }

    if (! isset($this->prepends[$section][$this->renderCount])) {
        $this->prepends[$section][$this->renderCount] = $content;
    } else {
        $this->prepends[$section][$this->renderCount] = $content.$this->prepends[$section][$this->renderCount];
    }
}
```

与`extendPush`不同的是，`extendPrepend`把`$content`连接到前面。