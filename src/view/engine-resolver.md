# 引擎解析器

引擎解析器负责按照给定的模板名称，推断出该模板适用的引擎。


```php
//src/Illuminate/View/Engines/EngineResolver.php

class EngineResolver
{
    /**
     * The array of engine resolvers.
     *
     * @var array
     */
    protected $resolvers = [];

    /**
     * The resolved engine instances.
     *
     * @var array
     */
    protected $resolved = [];

    /**
     * Register a new engine resolver.
     *
     * The engine string typically corresponds to a file extension.
     *
     * @param  string   $engine
     * @param  \Closure  $resolver
     * @return void
     */
    public function register($engine, Closure $resolver)
    {
        unset($this->resolved[$engine]);

        $this->resolvers[$engine] = $resolver;
    }

    /**
     * Resolve an engine instance by name.
     *
     * @param  string  $engine
     * @return \Illuminate\Contracts\View\Engine
     * @throws \InvalidArgumentException
     */
    public function resolve($engine)
    {
        if (isset($this->resolved[$engine])) {
            return $this->resolved[$engine];
        }

        if (isset($this->resolvers[$engine])) {
            return $this->resolved[$engine] = call_user_func($this->resolvers[$engine]);
        }

        throw new InvalidArgumentException("Engine [{$engine}] not found.");
    }
}
```

`register`负责注册引擎解析函数，实际上就是建立起引擎名称和对应的解析函数的映射，这样，在`resolve`方法中就能
按照给定的引擎名称，找到引擎解析函数，并且调用解析函数，最终拿到具体的引擎实例。