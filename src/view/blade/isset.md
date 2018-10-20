# isset

为了方便在模板中判断变量是否存在，Blade提供了`@isset`指令，用法如下：

```php
@isset($foo)
@else
@endisset
```

实现如下：


```php
//src/Illuminate/View/Compilers/Concerns/CompilesConditionals.php

/**
 * Compile the if-isset statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIsset($expression)
{
    return "<?php if(isset{$expression}): ?>";
}

/**
 * Compile the end-isset statements into valid PHP.
 *
 * @return string
 */
protected function compileEndIsset()
{
    return '<?php endif; ?>';
}
```