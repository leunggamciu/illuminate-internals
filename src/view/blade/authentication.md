# 认证

Blade支持在模板文件中判断用户是否登录，具体使用的指令为`@auth`和`@guest`，它们的用法如下：


```php
@auth('guardname')
@elseauth('guardname')
@endauth

@guest('guardname')
@elseguest('guardname')
@endguest
```

对应的实现为:


```php
//src/View/Compilers/Concerns/CompilesConditionals.php

/**
 * Compile the if-auth statements into valid PHP.
 *
 * @param  string|null  $guard
 * @return string
 */
protected function compileAuth($guard = null)
{
    $guard = is_null($guard) ? '()' : $guard;

    return "<?php if(auth()->guard{$guard}->check()): ?>";
}

/**
 * Compile the else-auth statements into valid PHP.
 *
 * @param  string|null  $guard
 * @return string
 */
protected function compileElseAuth($guard = null)
{
    $guard = is_null($guard) ? '()' : $guard;

    return "<?php elseif(auth()->guard{$guard}->check()): ?>";
}

/**
 * Compile the end-auth statements into valid PHP.
 *
 * @return string
 */
protected function compileEndAuth()
{
    return '<?php endif; ?>';
}

/**
 * Compile the if-guest statements into valid PHP.
 *
 * @param  string|null  $guard
 * @return string
 */
protected function compileGuest($guard = null)
{
    $guard = is_null($guard) ? '()' : $guard;

    return "<?php if(auth()->guard{$guard}->guest()): ?>";
}

/**
 * Compile the else-guest statements into valid PHP.
 *
 * @param  string|null  $guard
 * @return string
 */
protected function compileElseGuest($guard = null)
{
    $guard = is_null($guard) ? '()' : $guard;

    return "<?php elseif(auth()->guard{$guard}->guest()): ?>";
}

/**
 * Compile the end-guest statements into valid PHP.
 *
 * @return string
 */
protected function compileEndGuest()
{
    return '<?php endif; ?>';
}
```

这里的实现非常简单，就不再啰嗦了。