# 条件语句

Blade支持下面几种条件语句


```php
@if($condition)
@elseif()
@else
@endif


@unless($condition)
@elseif
@else
@endunless

@switch($val)
    @case(1)
        xxx
        @break
    @case(2)
        xxx
        @break
    @default
        xxx
@endswitch
```

它们都会被编译成对应的PHP条件语句，具体的实现过程如下：

```php


    /**
     * Compile the if statements into valid PHP.
     *
     * @param  string  $expression
     * @return string
     */
    protected function compileIf($expression)
    {
        return "<?php if{$expression}: ?>";
    }

    /**
     * Compile the unless statements into valid PHP.
     *
     * @param  string  $expression
     * @return string
     */
    protected function compileUnless($expression)
    {
        return "<?php if (! {$expression}): ?>";
    }

    /**
     * Compile the else-if statements into valid PHP.
     *
     * @param  string  $expression
     * @return string
     */
    protected function compileElseif($expression)
    {
        return "<?php elseif{$expression}: ?>";
    }

    /**
     * Compile the else statements into valid PHP.
     *
     * @return string
     */
    protected function compileElse()
    {
        return '<?php else: ?>';
    }

    /**
     * Compile the end-if statements into valid PHP.
     *
     * @return string
     */
    protected function compileEndif()
    {
        return '<?php endif; ?>';
    }

    /**
     * Compile the end-unless statements into valid PHP.
     *
     * @return string
     */
    protected function compileEndunless()
    {
        return '<?php endif; ?>';
    }


/**
 * Compile the switch statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileSwitch($expression)
{
    $this->firstCaseInSwitch = true;

    return "<?php switch{$expression}:";
}

/**
 * Compile the case statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileCase($expression)
{
    if ($this->firstCaseInSwitch) {
        $this->firstCaseInSwitch = false;

        return "case {$expression}: ?>";
    }

    return "<?php case {$expression}: ?>";
}

/**
 * Compile the default statements in switch case into valid PHP.
 *
 * @return string
 */
protected function compileDefault()
{
    return '<?php default: ?>';
}

/**
 * Compile the end switch statements into valid PHP.
 *
 * @return string
 */
protected function compileEndSwitch()
{
    return '<?php endswitch; ?>';
}
```

这里要注意的是`switch`语句，在`:`这种语法中，`switch`和第一个`case`之间不能有空格，所以`compileCase`里需要做判断，把`switch`
和第一个`case`包含在`<?php`和`?>`之间。
