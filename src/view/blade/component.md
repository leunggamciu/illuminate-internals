# 组件

在Blade中，我们可以使用`@component`指令来引用另外一个模板

```php
//widget.blade.php
<div>
    Widget content
    {{ $title }}
    {{ $slot }}
</div>
```

```php
@component('widget')
    @slot('title')
        Title
    @endslot
    custom content
@endcomponent
```

这样就能把widget这个view包含进来，类似`@include`一样。和`@include`不同的是，通过widget提供的slot，我们可以定制包含进来的view。

Blade把`@component`和`@endcomponent`指令编译成

```php
$__env->startComponent($expression);
$__env->renderComponent();
```

这两个方法对应的实现如下：

```php
//src/view/Concerns/ManagesComponents.php

/**
 * Start a component rendering process.
 *
 * @param  string  $name
 * @param  array  $data
 * @return void
 */
public function startComponent($name, array $data = [])
{
    if (ob_start()) {
        $this->componentStack[] = $name;

        $this->componentData[$this->currentComponent()] = $data;

        $this->slots[$this->currentComponent()] = [];
    }
}

/**
 * Render the current component.
 *
 * @return string
 */
public function renderComponent()
{
    $name = array_pop($this->componentStack);

    return $this->make($name, $this->componentData($name))->render();
}

/**
 * Get the data for the given component.
 *
 * @param  string  $name
 * @return array
 */
protected function componentData($name)
{
    return array_merge(
        $this->componentData[count($this->componentStack)],
        ['slot' => new HtmlString(trim(ob_get_clean()))],
        $this->slots[count($this->componentStack)]
    );
}

/**
 * Get the index for the current component.
 *
 * @return int
 */
protected function currentComponent()
{
    return count($this->componentStack) - 1;
}
```

`startComponent`主要开启输出缓冲，然后做一些初始化工作。`renderComponent`的实现也很简单，拿到当前渲染的组件的名称，
然后通过`make`来渲染这个组件。重点还是`componentData`，它把`@component`和`@endcomponent`之间的内容收集起来以后
然后与`slot`建立联系。

我们还能通过`@slot`和`@endslot`指令来定制指定的内容。它们会被Blade编译成

```php
$__env->slot($expression);
$__env->endSlot();
```

对应的实现如下：

```php
//src/view/Concerns/ManagesComponents.php

/**
 * Start the slot rendering process.
 *
 * @param  string  $name
 * @param  string|null  $content
 * @return void
 */
public function slot($name, $content = null)
{
    if (count(func_get_args()) == 2) {
        $this->slots[$this->currentComponent()][$name] = $content;
    } else {
        if (ob_start()) {
            $this->slots[$this->currentComponent()][$name] = '';

            $this->slotStack[$this->currentComponent()][] = $name;
        }
    }
}

/**
 * Save the slot content for rendering.
 *
 * @return void
 */
public function endSlot()
{
    last($this->componentStack);

    $currentSlot = array_pop(
        $this->slotStack[$this->currentComponent()]
    );

    $this->slots[$this->currentComponent()]
                [$currentSlot] = new HtmlString(trim(ob_get_clean()));
}
```

这两个方法的作用是把`@slot`和`@endslot`之间的内容之间的内容收集起来，然后放到`$this->slots`数组里，
供`componentData`方法来消费。

`slot`方法还告诉了我们一种在文档中没有提到的用法，那就是`@slot('title', 'this is a title')`，不需要
`@endslot`。