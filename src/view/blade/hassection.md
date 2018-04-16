# hassection

我们有时候需要判断一个section是否有内容，例如父页面允许子页面定制某一个section，但是当子页面没有定制内容的时候，那么父页面可以
有fallback的实现。

```php
//base.blade.php

@hassection('widget')
    @yield('widget')
@else
    base widget
@endif
```

`@hassection`指令的实现如下：


```php
//src/View/Compilers/Concerns/CompilesConditionals.php

/**
 * Compile the has-section statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileHasSection($expression)
{
    return "<?php if (! empty(trim(\$__env->yieldContent{$expression}))): ?>";
}
```

实际上就是判断给定section的内容是否为空。