# 助手函数

Blade支持一些助手函数指令：`@csrf`，`@dd($foo)`，`@dump($bar)`，`@method('DELETE')`

它们对应的用法可以参考文档，而对应的实现也很简单，都是在对应的助手函数基础上封装而成的。

```php
//src/Illuminate/View/Compilers/Concerns/CompilesHelpers.php

/**
 * Compile the CSRF statements into valid PHP.
 *
 * @return string
 */
protected function compileCsrf()
{
    return '<?php echo csrf_field(); ?>';
}

/**
 * Compile the "dd" statements into valid PHP.
 *
 * @param  string  $arguments
 * @return string
 */
protected function compileDd($arguments)
{
    return "<?php dd{$arguments}; ?>";
}

/**
 * Compile the "dump" statements into valid PHP.
 *
 * @param  string  $arguments
 * @return string
 */
protected function compileDump($arguments)
{
    return "<?php dump{$arguments}; ?>";
}

/**
 * Compile the method statements into valid PHP.
 *
 * @param  string  $method
 * @return string
 */
protected function compileMethod($method)
{
    return "<?php echo method_field{$method}; ?>";
}
```