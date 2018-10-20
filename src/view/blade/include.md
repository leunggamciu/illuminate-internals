# include

`@include`指令的实现很容易理解，它相当于渲染一遍include进来的模板，然后把渲染后的内容填充到模板中。

```php
//src/Illuminate/View/Compilers/Concerns/CompilesIncludes.php

/**
 * Compile the include statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileInclude($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->make({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}
```

它还有几个变种的指令，`@includeIf`，`@includeWhen`以及`@includeFirst`，对应的实现如下:


```php
//src/Illuminate/View/Compilers/Concerns/CompilesIncludes.php

/**
 * Compile the include-if statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeIf($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php if (\$__env->exists({$expression})) echo \$__env->make({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}

/**
 * Compile the include-when statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeWhen($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->renderWhen($expression, array_except(get_defined_vars(), array('__data', '__path'))); ?>";
}

/**
 * Compile the include-first statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileIncludeFirst($expression)
{
    $expression = $this->stripParentheses($expression);

    return "<?php echo \$__env->first({$expression}, array_except(get_defined_vars(), array('__data', '__path')))->render(); ?>";
}
```

具体的逻辑都下放到`$__env`对象里了。

`@includeIf`是在`@include`的基础上，先判断模板文件是否存在，存在的情况下才渲染。

```php
//src/Illuminate/View/Factory.php

/**
 * Determine if a given view exists.
 *
 * @param  string  $view
 * @return bool
 */
public function exists($view)
{
    try {
        $this->finder->find($view);
    } catch (InvalidArgumentException $e) {
        return false;
    }

    return true;
}
```

`@includeWhen`是在`@include`的基础上，先判断表达式的值是否为真。


```php
//src/Illuminate/View/Factory.php

/**
 * Get the rendered content of the view based on a given condition.
 *
 * @param  bool  $condition
 * @param  string  $view
 * @param  array   $data
 * @param  array   $mergeData
 * @return string
 */
public function renderWhen($condition, $view, $data = [], $mergeData = [])
{
    if (! $condition) {
        return '';
    }

    return $this->make($view, $this->parseData($data), $mergeData)->render();
}
```

`@includeFirst`是在`@include`的基础上，渲染最先找到的那个模板文件。

```php
//src/Illuminate/View/Factory.php

/**
 * Get the first view that actually exists from the given list.
 *
 * @param  array  $views
 * @param  array   $data
 * @param  array   $mergeData
 * @return \Illuminate\Contracts\View\View
 */
public function first(array $views, $data = [], $mergeData = [])
{
    $view = collect($views)->first(function ($view) {
        return $this->exists($view);
    });

    if (! $view) {
        throw new InvalidArgumentException('None of the views in the given array exist.');
    }

    return $this->make($view, $data, $mergeData);
}

```


Blade还支持一个结合了循环以及include的指令：`@each`，具体用法请参考文档。

它的实现如下：

```php
//src/Illuminate/View/Factory.php

/**
 * Get the rendered contents of a partial from a loop.
 *
 * @param  string  $view
 * @param  array   $data
 * @param  string  $iterator
 * @param  string  $empty
 * @return string
 */
public function renderEach($view, $data, $iterator, $empty = 'raw|')
{
    $result = '';

    // If is actually data in the array, we will loop through the data and append
    // an instance of the partial view to the final result HTML passing in the
    // iterated value of this data array, allowing the views to access them.
    if (count($data) > 0) {
        foreach ($data as $key => $value) {
            $result .= $this->make(
                $view, ['key' => $key, $iterator => $value]
            )->render();
        }
    }

    // If there is no data in the array, we will render the contents of the empty
    // view. Alternatively, the "empty view" could be a raw string that begins
    // with "raw|" for convenience and to let this know that it is a string.
    else {
        $result = Str::startsWith($empty, 'raw|')
                    ? substr($empty, 4)
                    : $this->make($empty)->render();
    }

    return $result;
}
```

它依次使用`$data`中的元素去渲染`$view`，然后把渲染后的结果拼接在一起。如果`$data`为空，那么
就渲染默认模板或者输出一段字符串。