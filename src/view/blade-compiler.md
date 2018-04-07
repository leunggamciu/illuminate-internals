# Blade模板编译器

Blade是一个模板编译器，它实现了`CompilerInterface`，这个接口要求编译器实现三个方法，一个是`getCompiledPath($path)`，用来获取
给定模板对应编译后文件的路径，一个是`isExpired($path)`，用来判断模板是否已经过期，要不要重新编译，最后一个是`compile($path)`，
它囊括了所有模板编译的操作。

Blade的`compile`实现很简短:


```php
//src/View/Compilers/BladeCompiler.php

/**
 * Compile the view at the given path.
 *
 * @param  string  $path
 * @return void
 */
public function compile($path = null)
{
    if ($path) {
        $this->setPath($path);
    }

    if (! is_null($this->cachePath)) {
        $contents = $this->compileString($this->files->get($this->getPath()));

        $this->files->put($this->getCompiledPath($this->getPath()), $contents);
    }
}
```

主要的实现逻辑就是先拿到模板文件的内容，编译后再写回编译结果文件中，真正的编译逻辑被下放到`compileString`中。


```php
//src/View/Compilers/BladeCompiler.php

/**
 * Compile the given Blade template contents.
 *
 * @param  string  $value
 * @return string
 */
public function compileString($value)
{
    if (strpos($value, '@verbatim') !== false) {
        $value = $this->storeVerbatimBlocks($value);
    }

    $this->footer = [];

    if (strpos($value, '@php') !== false) {
        $value = $this->storePhpBlocks($value);
    }

    $result = '';

    // Here we will loop through all of the tokens returned by the Zend lexer and
    // parse each one into the corresponding valid PHP. We will then have this
    // template as the correctly rendered PHP that can be rendered natively.
    foreach (token_get_all($value) as $token) {
        $result .= is_array($token) ? $this->parseToken($token) : $token;
    }

    if (! empty($this->rawBlocks)) {
        $result = $this->restoreRawContent($result);
    }

    // If there are any footer lines that need to get added to a template we will
    // add them here at the end of the template. This gets used mainly for the
    // template inheritance via the extends keyword that should be appended.
    if (count($this->footer) > 0) {
        $result = $this->addFooters($result);
    }

    return $result;
}
```

`compileString`先找到所有位于`@verbatim`,`@endverbatim`以及`@php`，`@endphp`之间的内容，然后把它们先存起来，
再用一个非PHP字符替换掉原来的PHP内容，等其他指令编译完成以后，再将之前的非PHP字符用之前存起来的内容替换掉。

这里多此一举的原因是，一方面，`@verbatim`块里可以包含Blade的指令字符，如果不把这些字符先替换掉，那么在后续的解析过程
中，这些指令就会被解析，这样就违反了`@verbatim`的语义了。另一方面，如果不用普通字符替换掉这些raw php代码的话，
那么`token_get_all`函数解析出来的结果就会非常多，因为会解析那些raw php代码，所以，为了性能考虑，只能减少`token_get_all`的结果，
所以把raw php代码替换成一个字符串，那么`token_get_all`作用在这些字符串时，只会产生一个**T_INLINE_HTML**token。

首先来看保存的逻辑

```php
//src/View/Compilers/BladeCompiler.php

/**
 * Store the verbatim blocks and replace them with a temporary placeholder.
 *
 * @param  string  $value
 * @return string
 */
protected function storeVerbatimBlocks($value)
{
    return preg_replace_callback('/(?<!@)@verbatim(.*?)@endverbatim/s', function ($matches) {
        return $this->storeRawBlock($matches[1]);
    }, $value);
}

/**
 * Store the PHP blocks and replace them with a temporary placeholder.
 *
 * @param  string  $value
 * @return string
 */
protected function storePhpBlocks($value)
{
    return preg_replace_callback('/(?<!@)@php(.*?)@endphp/s', function ($matches) {
        return $this->storeRawBlock("<?php{$matches[1]}?>");
    }, $value);
}
```

逻辑很简单，把匹配到的内容传递给`storeRawBlock`。


```php
//src/View/Compilers/BladeCompiler.php

/**
 * Store a raw block and return a unique raw placeholder.
 *
 * @param  string  $value
 * @return string
 */
protected function storeRawBlock($value)
{
    return $this->getRawPlaceholder(
        array_push($this->rawBlocks, $value) - 1
    );
}

/**
 * Get a placeholder to temporary mark the position of raw blocks.
 *
 * @param  int|string  $replace
 * @return string
 */
protected function getRawPlaceholder($replace)
{
    return str_replace('#', $replace, '@__raw_block_#__@');
}
```

`storeRawBlock`把传递进来的内容放到`$this->rawBlocks`中，然后拿到它的下标，然后转化成`@__raw_block_下标__@`格式的
字符串。

把存起来的raw php再拿出来放到原来的模板当中，是`restoreRawContent`的职责。


```php
//src/View/Compilers/BladeCompiler.php

/**
 * Replace the raw placeholders with the original code stored in the raw blocks.
 *
 * @param  string  $result
 * @return string
 */
protected function restoreRawContent($result)
{
    $result = preg_replace_callback('/'.$this->getRawPlaceholder('(\d+)').'/', function ($matches) {
        return $this->rawBlocks[$matches[1]];
    }, $result);

    $this->rawBlocks = [];

    return $result;
}
```

它的作用就是从`$this->rawBlocks`数组中拿到之前的存进去的raw php，然后根据`@__raw_block_下标__@`标识，替换到合适的位置。


## 输出

Blade支持三种输出指令：`{{ $val }}`，`{!! $val !!}`以及`@{{ $val }}`，前两者的区别是，前者会经过`htmlspecialchars`函数
的转换，而后者不会，第三个指令最终输出的结果只是把`@`符号拿掉，变成`{{ $val }}`。

编译输出指令的逻辑位于`src/Illuminate/View/Compilers/Concerns/CompilesEchos.php`，核心逻辑主要是以下4个方法：

```php
//src/Illuminate/View/Compilers/Concerns/CompilesEchos.php

/**
 * Compile the "raw" echo statements.
 *
 * @param  string  $value
 * @return string
 */
protected function compileRawEchos($value)
{
    $pattern = sprintf('/(@)?%s\s*(.+?)\s*%s(\r?\n)?/s', $this->rawTags[0], $this->rawTags[1]);

    $callback = function ($matches) {
        $whitespace = empty($matches[3]) ? '' : $matches[3].$matches[3];

        return $matches[1] ? substr($matches[0], 1) : "<?php echo {$this->compileEchoDefaults($matches[2])}; ?>{$whitespace}";
    };

    return preg_replace_callback($pattern, $callback, $value);
}

/**
 * Compile the "regular" echo statements.
 *
 * @param  string  $value
 * @return string
 */
protected function compileRegularEchos($value)
{
    $pattern = sprintf('/(@)?%s\s*(.+?)\s*%s(\r?\n)?/s', $this->contentTags[0], $this->contentTags[1]);

    $callback = function ($matches) {
        $whitespace = empty($matches[3]) ? '' : $matches[3].$matches[3];

        $wrapped = sprintf($this->echoFormat, $this->compileEchoDefaults($matches[2]));

        return $matches[1] ? substr($matches[0], 1) : "<?php echo {$wrapped}; ?>{$whitespace}";
    };

    return preg_replace_callback($pattern, $callback, $value);
}

/**
 * Compile the escaped echo statements.
 *
 * @param  string  $value
 * @return string
 */
protected function compileEscapedEchos($value)
{
    $pattern = sprintf('/(@)?%s\s*(.+?)\s*%s(\r?\n)?/s', $this->escapedTags[0], $this->escapedTags[1]);

    $callback = function ($matches) {
        $whitespace = empty($matches[3]) ? '' : $matches[3].$matches[3];

        return $matches[1] ? $matches[0] : "<?php echo e({$this->compileEchoDefaults($matches[2])}); ?>{$whitespace}";
    };

    return preg_replace_callback($pattern, $callback, $value);
}

/**
 * Compile the default values for the echo statement.
 *
 * @param  string  $value
 * @return string
 */
public function compileEchoDefaults($value)
{
    return preg_replace('/^(?=\$)(.+?)(?:\s+or\s+)(.+?)$/si', 'isset($1) ? $1 : $2', $value);
}
```


`compileRawEchos`用于编译`{{ $val }}`，把匹配到的花括号里面的内容传递给`compileEchoDefaults`方法，如果`{{ $val }}`前面带有`@`符号，
那么就直接返回`{{ $val }}`，这样也就实现了`@{{ $val }}`的语义了。`compileEchoDefaults`的实现也很简单，通过观察这个方法，还可以发现一种
文档中并没有提及的写法：`{{ $val or '$val not set' }}`，就是当`$val`不存在时候输出别的字符。


## 注释

Blade支持注释，语法为`{{-- comment!!! --}}`，对应的实现也很简单，就是用正则表达式匹配，然后替换为空字符串


```php
//src/View/Compilers/Concerns/CompilesComments.php

/**
 * Compile Blade comments into an empty string.
 *
 * @param  string  $value
 * @return string
 */
protected function compileComments($value)
{
    $pattern = sprintf('/%s--(.*?)--%s/s', $this->contentTags[0], $this->contentTags[1]);

    return preg_replace($pattern, '', $value);
}
```

## PHP

Blade通过`@php`以及`@endphp`指令来支持在模板中写PHP代码。只需要把PHP代码放到`@php`和`@endphp`之间就可以了。


```
@php
echo "hello world";
@endphp
```

`@php`还支持传入PHP表达式，`@php ($a = 1)`，这时候就不需要`@endphp`了，它会被编译成`<?php $a = 1; ?>`。

`@php`块的实现在前面已经解释过了，这里主要是解释`@php`表达式


```php
//src/View/Compilers/Concerns/CompilesRawPhp.php

/**
 * Compile the raw PHP statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compilePhp($expression)
{
    if ($expression) {
        return "<?php {$expression}; ?>";
    }

    return '@php';
}
```

十分简单，只是用`<?php ?>`标签包围起来。在同一个trait里面，还有一个方法:

```php
//src/View/Compilers/Concerns/CompilesRawPhp.php

/**
 * Compile the unset statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileUnset($expression)
{
    return "<?php unset{$expression}; ?>";
}
```

也是很简单，只调用了`unset`函数。所以在模板下可以用`@unset`指令来实现unset。

## include

`@include`指令的实现很容易理解，它相当于渲染一遍include进来的模板，然后把渲染后的内容填充到模板中。

```php
//src/View/Compilers/Concerns/CompilesIncludes.php

/**
 * Compile the include statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileInclude($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->make({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}
```

它还有几个变种的指令，`@includeIf`，`@includeWhen`以及`@includeFirst`，对应的实现如下:


```php
//src/View/Compilers/Concerns/CompilesIncludes.php

/**
 * Compile the include-if statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeIf($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php if (\$__env->exists({$expression})) echo \$__env->make({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}

/**
 * Compile the include-when statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeWhen($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->renderWhen($expression, array_except(get_defined_vars(), array('__data', '__path'))); ?>";
}

/**
 * Compile the include-first statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeFirst($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->first({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}
```

具体的逻辑都下放到`$__env`对象里了。

`@includeIf`是在`@include`的基础上，先判断模板文件是否存在，存在的情况下才渲染。

```php
//src/View/Factory.php

/**
 * Determine if a given view exists.
 *
 * @param  string  $view
 * @return bool
 */
public function exists($view)
{
    try {
        $this->finder->find($view);
    } catch (InvalidArgumentException $e) {
        return false;
    }

    return true;
}
```

`@includeWhen`是在`@include`的基础上，先判断表达式的值是否为真。


```php
//src/View/Factory.php

/**
 * Get the rendered content of the view based on a given condition.
 *
 * @param  bool  $condition
 * @param  string  $view
 * @param  array   $data
 * @param  array   $mergeData
 * @return string
 */
public function renderWhen($condition, $view, $data = [], $mergeData = [])
{
    if (! $condition) {
        return '';
    }

    return $this->make($view, $this->parseData($data), $mergeData)->render();
}
```

`@includeFirst`是在`@include`的基础上，渲染最先找到的那个模板文件。

```php
//src/View/Factory.php

/**
 * Get the first view that actually exists from the given list.
 *
 * @param  array  $views
 * @param  array   $data
 * @param  array   $mergeData
 * @return \Illuminate\Contracts\View\View
 */
public function first(array $views, $data = [], $mergeData = [])
{
    $view = collect($views)->first(function ($view) {
        return $this->exists($view);
    });

    if (! $view) {
        throw new InvalidArgumentException('None of the views in the given array exist.');
    }

    return $this->make($view, $data, $mergeData);
}

```


Blade还支持一个结合了循环以及include的指令：`@each`，具体用法请参考文档。

它的实现如下：

```php
//src/View/Factory.php

/**
 * Get the rendered contents of a partial from a loop.
 *
 * @param  string  $view
 * @param  array   $data
 * @param  string  $iterator
 * @param  string  $empty
 * @return string
 */
public function renderEach($view, $data, $iterator, $empty = 'raw|')
{
    $result = '';

    // If is actually data in the array, we will loop through the data and append
    // an instance of the partial view to the final result HTML passing in the
    // iterated value of this data array, allowing the views to access them.
    if (count($data) > 0) {
        foreach ($data as $key => $value) {
            $result .= $this->make(
                $view, ['key' => $key, $iterator => $value]
            )->render();
        }
    }

    // If there is no data in the array, we will render the contents of the empty
    // view. Alternatively, the "empty view" could be a raw string that begins
    // with "raw|" for convenience and to let this know that it is a string.
    else {
        $result = Str::startsWith($empty, 'raw|')
                    ? substr($empty, 4)
                    : $this->make($empty)->render();
    }

    return $result;
}
```

它依次使用`$data`中的元素去渲染`$view`，然后把渲染后的结果拼接在一起。如果`$data`为空，那么
就渲染默认模板或者输出一段字符串。


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

$__env->startSection('title');
Title from page
$__env->stopSection();

echo $__env->make('base', array_except(get_defined_vars(), array('__data', '__path')))->render();
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
//src/View/Concerns/ManagesLayouts.php

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
//src/View/Concerns/ManagesLayouts.php

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

## stack

在Blade中，我们可以使用`@stack`指令定义可以占位符，然后用`@push`和`@prepend`指令往占位符里添加内容。更具体的用法请参考文档。

`@stack`指令经过编译后，变成了

```php
<?php echo $__env->yieldPushContent($expression); ?>
```

下面是`yieldPushContent`的实现

```php
//src/View/Concerns/ManagesStacks.php


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
//src/View/Concerns/ManagesStacks.php

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
//src/View/Concerns/ManagesStacks.php

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