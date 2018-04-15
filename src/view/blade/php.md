# PHP

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