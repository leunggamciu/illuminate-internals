# 服务发现

在实现服务发现之前，服务提供者需要手动添加到`config/app.php`中，而利用服务发现机制，服务提供者的作者可以通过`composer.json`的
`extra`字段，将服务提供者注册到Laravel应用中。这样，用户就不再需要手动添加服务提供者，只需要像安装其他composer包一样，安装
服务提供者。

`\Illuminate\Foundation\PackageManifest`就是负责搜集所有通过composer安装的服务提供者。首先看看它的构造方法

```php
//src/Illuminate/Foundation/PackageManifest.php

/**
 * Create a new package manifest instance.
 *
 * @param  \Illuminate\Filesystem\Filesystem  $files
 * @param  string  $basePath
 * @param  string  $manifestPath
 * @return void
 */
public function __construct(Filesystem $files, $basePath, $manifestPath)
{
    $this->files = $files;
    $this->basePath = $basePath;
    $this->manifestPath = $manifestPath;
    $this->vendorPath = $basePath.'/vendor';
}
```

这里要留意的就是`$manifestPath`，Laravel将搜集到的通过composer安装的服务提供者都缓存到一个文件中，`$manifestPath`就是这个文件的路径。

```php
//src/Illuminate/Foundation/PackageManifest.php

/**
 * Get all of the service provider class names for all packages.
 *
 * @return array
 */
public function providers()
{
    return collect($this->getManifest())->flatMap(function ($configuration) {
        return (array) ($configuration['providers'] ?? []);
    })->filter()->all();
}

/**
 * Get the current package manifest.
 *
 * @return array
 */
protected function getManifest()
{
    if (! is_null($this->manifest)) {
        return $this->manifest;
    }

    if (! file_exists($this->manifestPath)) {
        $this->build();
    }

    return $this->manifest = file_exists($this->manifestPath) ?
        $this->files->getRequire($this->manifestPath) : [];
}
```

`providers`提供搜集到的所有服务提供者，`getManifest`的实现很简单，这要看下`build`方法

```php
//src/Illuminate/Foundation/PackageManifest.php

/**
 * Build the manifest and write it to disk.
 *
 * @return void
 */
public function build()
{
    $packages = [];

    if ($this->files->exists($path = $this->vendorPath.'/composer/installed.json')) {
        $packages = json_decode($this->files->get($path), true);
    }

    $ignoreAll = in_array('*', $ignore = $this->packagesToIgnore());

    $this->write(collect($packages)->mapWithKeys(function ($package) {
        return [$this->format($package['name']) => $package['extra']['laravel'] ?? []];
    })->each(function ($configuration) use (&$ignore) {
        $ignore = array_merge($ignore, $configuration['dont-discover'] ?? []);
    })->reject(function ($configuration, $package) use ($ignore, $ignoreAll) {
        return $ignoreAll || in_array($package, $ignore);
    })->filter()->all());
}
```

应用中安装的所有composer依赖信息都放到了`vendor/composer/installed.json`这个文件中，它通过迭代这个json文件的内容
搜集所有服务提供者，并过滤掉那些`dont-discover`的服务提供者，最后形成一个数组，通过`write`方法写入到缓存文件中。

```php
//src/Illuminate/Foundation/PackageManifest.php

/**
 * Write the given manifest array to disk.
 *
 * @param  array  $manifest
 * @return void
 * @throws \Exception
 */
protected function write(array $manifest)
{
    if (! is_writable(dirname($this->manifestPath))) {
        throw new Exception('The '.dirname($this->manifestPath).' directory must be present and writable.');
    }

    $this->files->put(
        $this->manifestPath, '<?php return '.var_export($manifest, true).';'
    );
}
```