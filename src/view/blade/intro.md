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