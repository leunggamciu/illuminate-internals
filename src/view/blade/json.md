# JSON

Blade支持`@json`指令，将PHP数组转换为JSON字符串，用法如下:

```php
@json(['foo' => 'bar'])
```

对应的实现如下：


```php
//src/Illuminate/View/Compilers/Concerns/CompilersJson.php

/**
 * Compile the JSON statement into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileJson($expression)
{
    $parts = explode(',', $this->stripParentheses($expression));

    $options = isset($parts[1]) ? trim($parts[1]) : $this->encodingOptions;

    $depth = isset($parts[2]) ? trim($parts[2]) : 512;

    return "<?php echo json_encode($parts[0], $options, $depth) ?>";
}
```

从`compileJson`可以看出文档未记录的用法：

```php
@json(['foo' => 'bar'], JSON_HEX_TAG, 512)
```

后面两个参数对应的是`json_encode`函数后面的两个参数