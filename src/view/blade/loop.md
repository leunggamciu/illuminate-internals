# 循环语句

Blade支持多种循环语句，首先是普通的`@foreach`语句：

```php
@foreach($users as $user)

@endforeach
```

对应的实现是

```php
//src/Illuminate/View/Compilers/Concerns/CompilesLoops.php

/**
 * Compile the for-each statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileForeach($expression)
{
    preg_match('/\( *(.*) +as *(.*)\)$/is', $expression, $matches);

    $iteratee = trim($matches[1]);

    $iteration = trim($matches[2]);

    $initLoop = "\$__currentLoopData = {$iteratee}; \$__env->addLoop(\$__currentLoopData);";

    $iterateLoop = '$__env->incrementLoopIndices(); $loop = $__env->getLastLoop();';

    return "<?php {$initLoop} foreach(\$__currentLoopData as {$iteration}): {$iterateLoop} ?>";
}


/**
 * Compile the end-for-each statements into valid PHP.
 *
 * @return string
 */
protected function compileEndforeach()
{
    return '<?php endforeach; $__env->popLoop(); $loop = $__env->getLastLoop(); ?>';
}
```

经过`compileForeach`和`compileEndforeach`方法以后，最终在模板中生成下面的字符串：


```php

$__currentLoopData = $users;
$__env->addLoop($__currentLoopData);
foreach ($__currentLoopData as $user):
    $__env->incrementLoopIndices();
    $loop = $__env->getLastLoop();

    //循环体
endforeach;
$__env->popLoop();
$loop = $__env->getLastLoop();
```


经过这样转换以后，在模板中，在`@foreach`循环体内，就能使用`$loop`这个变量了，通过`$loop`，我们能知道当前处于第几次迭代、是不是第一次迭代、
是不是最后一次迭代、还剩下多少次迭代、嵌套循环的深度、以及访问父循环中的`$loop`变量。

先来看看`addLoop`的实现


```php
//src/Illuminate/View/Concerns/ManageLoops.php

/**
 * Add new loop to the stack.
 *
 * @param  \Countable|array  $data
 * @return void
 */
public function addLoop($data)
{
    $length = is_array($data) || $data instanceof Countable ? count($data) : null;

    $parent = Arr::last($this->loopsStack);

    $this->loopsStack[] = [
        'iteration' => 0,
        'index' => 0,
        'remaining' => $length ?? null,
        'count' => $length,
        'first' => true,
        'last' => isset($length) ? $length == 1 : null,
        'depth' => count($this->loopsStack) + 1,
        'parent' => $parent ? (object) $parent : null,
    ];
}
```

它接受当前需要迭代的变量，然后将计算好各个值放到数组中，并推入`$this->loopStack`。当进入循环体以后，就开始变更各个数值了。
变更的过程由`incrementLoopIndices`方法实现。


```php
//src/Illuminate/View/Concerns/ManageLoops.php

/**
 * Increment the top loop's indices.
 *
 * @return void
 */
public function incrementLoopIndices()
{
    $loop = $this->loopsStack[$index = count($this->loopsStack) - 1];

    $this->loopsStack[$index] = array_merge($this->loopsStack[$index], [
        'iteration' => $loop['iteration'] + 1,
        'index' => $loop['iteration'],
        'first' => $loop['iteration'] == 0,
        'remaining' => isset($loop['count']) ? $loop['remaining'] - 1 : null,
        'last' => isset($loop['count']) ? $loop['iteration'] == $loop['count'] - 1 : null,
    ]);
}
```

除了`@foreach`以外，Blade还支持`@forelse`指令，`@forelse`结合了`@foreach`以及条件判断语句。

```php
@forelse($users as $user)
    {{ $user->name }}
@empty
    empty
@endforelse
```

它最终编译成


```php
$__empty_1 = true;
$__currentLoopData = $users;
$__env->addLoop($__currentLoopData);
foreach ($__currentLoopData as $user):
    $__env->incrementLoopIndices();
    $loop = $__env->getLastLoop();
    $__empty_1 = false;

    //循环体
endforeach;
$__env->popLoop();
$loop = $__env->getLastLoop();

if ($__empty_1):
    //如果为空
endif;
```

与`@foreach`十分相似，只不过多了个`$__empty_1`变量，用来跟踪是否为空，一旦进入循环体，那么`$__empty_1`就置为假。


以下是`@forelse`的实现

```php
//src/Illuminate/View/Concerns/ManageLoops.php

/**
 * Counter to keep track of nested forelse statements.
 *
 * @var int
 */
protected $forElseCounter = 0;

/**
 * Compile the for-else statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileForelse($expression)
{
    $empty = '$__empty_'.++$this->forElseCounter;

    preg_match('/\( *(.*) +as *(.*)\)$/is', $expression, $matches);

    $iteratee = trim($matches[1]);

    $iteration = trim($matches[2]);

    $initLoop = "\$__currentLoopData = {$iteratee}; \$__env->addLoop(\$__currentLoopData);";

    $iterateLoop = '$__env->incrementLoopIndices(); $loop = $__env->getLastLoop();';

    return "<?php {$empty} = true; {$initLoop} foreach(\$__currentLoopData as {$iteration}): {$iterateLoop} {$empty} = false; ?>";
}

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
 * Compile the end-for-else statements into valid PHP.
 *
 * @return string
 */
protected function compileEndforelse()
{
    return '<?php endif; ?>';
}
```

Blade还支持`@break`和`@continue`指令


```php
//src/Illuminate/View/Concerns/ManageLoops.php

/**
 * Compile the break statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileBreak($expression)
{
    if ($expression) {
        preg_match('/\(\s*(-?\d+)\s*\)$/', $expression, $matches);

        return $matches ? '<?php break '.max(1, $matches[1]).'; ?>' : "<?php if{$expression} break; ?>";
    }

    return '<?php break; ?>';
}

/**
 * Compile the continue statements into valid PHP.
 *
 * @param  string  $expression
 * @return string
 */
protected function compileContinue($expression)
{
    if ($expression) {
        preg_match('/\(\s*(-?\d+)\s*\)$/', $expression, $matches);

        return $matches ? '<?php continue '.max(1, $matches[1]).'; ?>' : "<?php if{$expression} continue; ?>";
    }

    return '<?php continue; ?>';
}
```

它们的用法分为三种情况，`@break`和`@continue`和`break`以及`continue`的语义一致，当使用`@break(2)`或者`@continue(2)`，也就是
将数字作为参数时，语义就和`break 2`以及`continue 2`一致，而当使用`@break(true)`或者`@continue(true)`的时候，它们接受一个表达式，
只有表达式的值为真是，才执行对应的`break`或者`continue`操作。

除了前面两种循环语句，Blade也支持`@for`和`@while`，对应的实现很简单，就不再叙述了。