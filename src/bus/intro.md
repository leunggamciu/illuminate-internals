# 总线

总线是Laravel中的一个基础组件，它实现的是设计模式中的[指令（命令）模式](https://designpatternsphp.readthedocs.io/en/latest/Behavioral/Command/README.html)。通过总线，我们可以分发同步指令和异步指令。在Laravel中，指令就是一个具有`handle`方法的类

```php
class SendEmailCommand
{
    public function handle()
    {

    }
}

//分发同步指令
$bus->dispatch(new SendEmailCommand);
```

上面是一段分发同步指令的代码，而要把同步指令变成异步指令，只需要实现`\Illuminate\Contracts\Queue\ShouldQueue`接口。

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class SendEmailCommand implements ShouldQueue
{
    public function handle()
    {

    }
}

//分发异步指令
$bus->dispatch(new SendEmailCommand);
```

异步指令在Laravel的术语里面也叫作业(job)。注意，这里先不考虑自分发(self dispatching)，自定义队列以及异步指令中含有Eloquent模型的情况。