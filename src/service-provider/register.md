# 服务注册

应用在启动阶段通过使用`\Illuminate\Foundation\Application`中的`registerConfiguredProviders`注册所有服务提供者。

```php
//src/Illuminate/Foundation/Application.php

/**
 * Register all of the configured providers.
 *
 * @return void
 */
public function registerConfiguredProviders()
{
    $providers = Collection::make($this->config['app.providers'])
                    ->partition(function ($provider) {
                        return Str::startsWith($provider, 'Illuminate\\');
                    });

    $providers->splice(1, 0, [$this->make(PackageManifest::class)->providers()]);

    (new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath()))
                ->load($providers->collapse()->toArray());
}
```

它将`config/app.php`中的服务提供者以及通过composer安装的服务提供者搜集起来，然后通过`\Illuminate\Foundation\ProviderRepository`中的
`load`方法，将这些服务提供者加载进来。

```php
//src/Illuminate/Foundation/ProviderRepository.php

/**
 * Register the application service providers.
 *
 * @param  array  $providers
 * @return void
 */
public function load(array $providers)
{
    $manifest = $this->loadManifest();

    // First we will load the service manifest, which contains information on all
    // service providers registered with the application and which services it
    // provides. This is used to know which services are "deferred" loaders.
    if ($this->shouldRecompile($manifest, $providers)) {
        $manifest = $this->compileManifest($providers);
    }

    // Next, we will register events to load the providers for each of the events
    // that it has requested. This allows the service provider to defer itself
    // while still getting automatically loaded when a certain event occurs.
    foreach ($manifest['when'] as $provider => $events) {
        $this->registerLoadEvents($provider, $events);
    }

    // We will go ahead and register all of the eagerly loaded providers with the
    // application so their services can be registered with the application as
    // a provided service. Then we will set the deferred service list on it.
    foreach ($manifest['eager'] as $provider) {
        $this->app->register($provider);
    }

    $this->app->addDeferredServices($manifest['deferred']);
}
```

`\Illuminate\Foundation\ProviderRepository`本身也维护着一个缓存文件，它里面包含当前应用使用的所有服务提供者，并且按照即时服务提供者，
条件服务提供者，延迟服务提供者分好类。在一开始，它会通过这个缓存文件来加载应用中所有服务提供者，如果这个文件已经无效了，那么就重新生成一遍。

在获取到所有服务提供者以后，就开始注册的逻辑

首先注册的是条件服务提供者

```php
//src/Illuminate/Foundation/ProviderRepository.php

/**
 * Register the load events for the given provider.
 *
 * @param  string  $provider
 * @param  array  $events
 * @return void
 */
protected function registerLoadEvents($provider, array $events)
{
    if (count($events) < 1) {
        return;
    }

    $this->app->make('events')->listen($events, function () use ($provider) {
        $this->app->register($provider);
    });
}
```

实际上，它就是监听服务提供者中`when`方法返回的所有事件，当注册事件触发的时候，就使用`\Illuminate\Foundation\Application`的`register`
方法来注册该服务提供者。接着是注册即时服务提供者，它就是简单地调用注册方法，完成注册。

```php
//src/Illuminate/Foundation/Application.php

/**
 * Add an array of services to the application's deferred services.
 *
 * @param  array  $services
 * @return void
 */
public function addDeferredServices(array $services)
{
    $this->deferredServices = array_merge($this->deferredServices, $services);
}
```

最后就是把延迟服务提供者先保存起来，先不执行注册流程。

接下来看具体的注册流程

```php
//src/Illuminate/Foundation/Application.php

/**
 * Register a service provider with the application.
 *
 * @param  \Illuminate\Support\ServiceProvider|string  $provider
 * @param  array  $options
 * @param  bool   $force
 * @return \Illuminate\Support\ServiceProvider
 */
public function register($provider, $options = [], $force = false)
{
    if (($registered = $this->getProvider($provider)) && ! $force) {
        return $registered;
    }

    // If the given "provider" is a string, we will resolve it, passing in the
    // application instance automatically for the developer. This is simply
    // a more convenient way of specifying your service provider classes.
    if (is_string($provider)) {
        $provider = $this->resolveProvider($provider);
    }

    if (method_exists($provider, 'register')) {
        $provider->register();
    }

    // If there are bindings / singletons set as properties on the provider we
    // will spin through them and register them with the application, which
    // serves as a convenience layer while registering a lot of bindings.
    if (property_exists($provider, 'bindings')) {
        foreach ($provider->bindings as $key => $value) {
            $this->bind($key, $value);
        }
    }

    if (property_exists($provider, 'singletons')) {
        foreach ($provider->singletons as $key => $value) {
            $this->singleton($key, $value);
        }
    }

    $this->markAsRegistered($provider);

    // If the application has already booted, we will call this boot method on
    // the provider class so it has an opportunity to do its boot logic and
    // will be ready for any usage by this developer's application logic.
    if ($this->booted) {
        $this->bootProvider($provider);
    }

    return $provider;
}
```

首先如果已经注册了，那么就不再执行注册流程，直接返回。如果没有注册，那么就

* 创建服务提供者
* 调用服务提供者的register方法
* 如果服务提供者有`$bindings`属性，那么就开始注册这些绑定
* 如果服务提供者有`$singleton`属性，那么就开始注册单例
* 将服务提供者标记为已注册
* 如果应用已经启动了，那么就调用服务提供者的boot方法

这里要注意的是有一段启动服务提供者的逻辑，这是因为有一些服务提供者可能会在应用启动了以后才注册的，譬如条件服务提供者，你不知道触发事件的那一刻
应用到底是处于已经启动还是未启动的状态。

延迟服务提供者就是在使用的时候才执行注册流程，服务提供者中的`provides`方法就是返回要使用的容器绑定。通过，我们通过容器中的`make`方法
来获取容器中的绑定，这时候就是我们使用的时候。

```php
//src/Illuminate/Foundation/Application.php

/**
 * Resolve the given type from the container.
 *
 * (Overriding Container::make)
 *
 * @param  string  $abstract
 * @param  array  $parameters
 * @return mixed
 */
public function make($abstract, array $parameters = [])
{
    $abstract = $this->getAlias($abstract);

    if (isset($this->deferredServices[$abstract]) && ! isset($this->instances[$abstract])) {
        $this->loadDeferredProvider($abstract);
    }

    return parent::make($abstract, $parameters);
}
```

当我们使用的绑定是一个延迟服务时，它就会调用`loadDeferredProvider`来加载该服务

```php
//src/Illuminate/Foundation/Application.php

/**
 * Load and boot all of the remaining deferred providers.
 *
 * @return void
 */
public function loadDeferredProviders()
{
    // We will simply spin through each of the deferred providers and register each
    // one and boot them if the application has booted. This should make each of
    // the remaining services available to this application for immediate use.
    foreach ($this->deferredServices as $service => $provider) {
        $this->loadDeferredProvider($service);
    }

    $this->deferredServices = [];
}

/**
 * Load the provider for a deferred service.
 *
 * @param  string  $service
 * @return void
 */
public function loadDeferredProvider($service)
{
    if (! isset($this->deferredServices[$service])) {
        return;
    }

    $provider = $this->deferredServices[$service];

    // If the service provider has not already been loaded and registered we can
    // register it with the application and remove the service from this list
    // of deferred services, since it will already be loaded on subsequent.
    if (! isset($this->loadedProviders[$provider])) {
        $this->registerDeferredProvider($provider, $service);
    }
}

/**
 * Register a deferred provider and service.
 *
 * @param  string  $provider
 * @param  string|null  $service
 * @return void
 */
public function registerDeferredProvider($provider, $service = null)
{
    // Once the provider that provides the deferred service has been registered we
    // will remove it from our local list of the deferred services with related
    // providers so that this container does not try to resolve it out again.
    if ($service) {
        unset($this->deferredServices[$service]);
    }

    $this->register($instance = new $provider($this));

    if (! $this->booted) {
        $this->booting(function () use ($instance) {
            $this->bootProvider($instance);
        });
    }
}
```

这里要留意的就是`registerDeferredProvider`，在使用服务的时候，如果应用还没有启动，那么就要监听应用启动
事件，在应用启动的时候，再启动这个服务提供者。