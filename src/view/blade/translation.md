# translation

我们能在Blade中使用`@lang`指令来实现字符串的本地化


```php
@lang('auth.failed')

@lang
auth.failed
@endlang

@lang(['key' => 'value'])
auth.failed
@endlang
```

它们会被编译成

```php
//src/Illuminate/View/Compilers/Concerns/CompilesTranslations.php

/**
 * Compile the lang statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileLang($expression)
{
    if (is_null($expression)) {
        return '<?php $__env->startTranslation(); ?>';
    } elseif ($expression[1] === '[') {
        return "<?php \$__env->startTranslation{$expression}; ?>";
    }

    return "<?php echo app('translator')->getFromJson{$expression}; ?>";
}

/**
 * Compile the end-lang statements into valid PHP.
 *
 * @return string
 */
protected function compileEndlang()
{
    return '<?php echo $__env->renderTranslation(); ?>';
}
```

`startTranslation`方法主要是开启输出缓冲，对应的，`renderTranslation`则是取出缓冲区的内容，然后关闭缓冲区。
把缓冲区的内容，也就是处于`@lang`和`@endlang`之间的内容，传递给translator的`getFromJson`方法。

```php
//src/Illuminate/View/Concerns/ManagesTranslations.php

/**
 * Start a translation block.
 *
 * @param  array  $replacements
 * @return void
 */
public function startTranslation($replacements = [])
{
    ob_start();

    $this->translationReplacements = $replacements;
}

/**
 * Render the current translation.
 *
 * @return string
 */
public function renderTranslation()
{
    return $this->container->make('translator')->getFromJson(
        trim(ob_get_clean()), $this->translationReplacements
    );
}
```