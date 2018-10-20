# empty

Blade支持`@empty`指令，也就是只有在参数为空的情况下，才执行对应的逻辑。

```php
@empty($user)
    xxx
@endempty
```

它们会被编译成


```php
if (empty($user)):
    xxx
endif;
```

下面为对应的实现


```php
//src/Illuminate/View/Compilers/Concerns/CompilesLoops.php

/**
 * Compile the for-else-empty and empty statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileEmpty($expression)
{
    if ($expression) {
        return "<?php if(empty{$expression}): ?>";
    }

    $empty = '$__empty_'.$this->forElseCounter--;

    return "<?php endforeach; \$__env->popLoop(); \$loop = \$__env->getLastLoop(); if ({$empty}): ?>";
}

/**
 * Compile the end-empty statements into valid PHP.
 *
 * @return string
 */
protected function compileEndEmpty()
{
    return '<?php endif; ?>';
}
```