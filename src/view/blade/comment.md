# 注释

Blade支持注释，语法为`{{-- comment!!! --}}`，对应的实现也很简单，就是用正则表达式匹配，然后替换为空字符串


```php
//src/View/Compilers/Concerns/CompilesComments.php

/**
 * Compile Blade comments into an empty string.
 *
 * @param  string  $value
 * @return string
 */
protected function compileComments($value)
{
    $pattern = sprintf('/%s--(.*?)--%s/s', $this->contentTags[0], $this->contentTags[1]);

    return preg_replace($pattern, '', $value);
}
```