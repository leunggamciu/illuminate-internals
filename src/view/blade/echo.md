# 输出 

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