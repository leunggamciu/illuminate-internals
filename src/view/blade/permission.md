# 权限

Blade支持在模板文件中判断用户是否具有某些权限，所用到的指令有`@can`和`@cannot`，用法如下：

```php
@can('update', $post)
@elsecan('create', App\Post::class)
@endcan

@cannot('update', $post)
@elsecannot('create', App\Post::class)
@endcannot
```

具体的实现如下：

```php
//src/Illuminate/View/Compilers/Concerns/CompilesAuthorizations.php

/**
 * Compile the can statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileCan($expression)
{
    return "<?php if (app(\Illuminate\\Contracts\\Auth\\Access\\Gate::class)->check{$expression}): ?>";
}

/**
 * Compile the cannot statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileCannot($expression)
{
    return "<?php if (app(\Illuminate\\Contracts\\Auth\\Access\\Gate::class)->denies{$expression}): ?>";
}

/**
 * Compile the else-can statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileElsecan($expression)
{
    return "<?php elseif (app(\Illuminate\\Contracts\\Auth\\Access\\Gate::class)->check{$expression}): ?>";
}

/**
 * Compile the else-cannot statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileElsecannot($expression)
{
    return "<?php elseif (app(\Illuminate\\Contracts\\Auth\\Access\\Gate::class)->denies{$expression}): ?>";
}

/**
 * Compile the end-can statements into valid PHP.
 *
 * @return string
 */
protected function compileEndcan()
{
    return '<?php endif; ?>';
}

/**
 * Compile the end-cannot statements into valid PHP.
 *
 * @return string
 */
protected function compileEndcannot()
{
    return '<?php endif; ?>';
}
```

具体的逻辑都放到了`Gate`这个类里面，这里先不作分析。