# view文件查找器

view文件查找器主要负责通过给定的view名称，从指定的候选路径中，找到对应的view文件。这样我们在应用层上就不需要
指定view的完整路径了。

首先来看看查找器的构造方法

```php
//src/Illuminate/View/FileViewFinder.php

/**
 * Create a new file view loader instance.
 *
 * @param  \Illuminate\Filesystem\Filesystem  $files
 * @param  array  $paths
 * @param  array  $extensions
 * @return void
 */
public function __construct(Filesystem $files, array $paths, array $extensions = null)
{
    $this->files = $files;
    $this->paths = $paths;

    if (isset($extensions)) {
        $this->extensions = $extensions;
    }
}
```

第一个参数是一个操作文件系统的对象，通过它能判断文件是否存在等，第二个参数是候选路径集合，查找器就是从这些候选路径
中查找，第三个参数是view文件的后缀名，只有满足这些后缀名的文件才有可能被查找到，它预先设置了三种后缀名称：
`blade.php`，`php`以及`css`。

接着是`find`方法

```php
//src/Illuminate/View/FileViewFinder.php

/**
 * Get the fully qualified location of the view.
 *
 * @param  string  $name
 * @return string
 */
public function find($name)
{
    if (isset($this->views[$name])) {
        return $this->views[$name];
    }

    if ($this->hasHintInformation($name = trim($name))) {
        return $this->views[$name] = $this->findNamespacedView($name);
    }

    return $this->views[$name] = $this->findInPaths($name, $this->paths);
}
```

查找器的查找功能就由它实现，逻辑很简单，如果之前有查找过的，那么直接返回之前找到的路径名；如果是view名称带有命名空间(也就是view名称前面还有个`::`)
那么就从候选的命名空间中查找；最后就是普通的查找。

命名空间的作用是使得第三方package的view也能被查找到，而不需要将所有view文件集中到一个目录中。查找器内部维护一个从命名空间到其候选路径集合的映射，
当view名称带有命名空间的情况下，那么就从该命名空间下的候选路径集合中查找。

下面是更加具体的实现

```php
//src/Illuminate/View/FileViewFinder.php

/**
 * Get the path to a template with a named path.
 *
 * @param  string  $name
 * @return string
 */
protected function findNamespacedView($name)
{
    list($namespace, $view) = $this->parseNamespaceSegments($name);

    return $this->findInPaths($view, $this->hints[$namespace]);
}

/**
 * Find the given view in the list of paths.
 *
 * @param  string  $name
 * @param  array   $paths
 * @return string
 *
 * @throws \InvalidArgumentException
 */
protected function findInPaths($name, $paths)
{
    foreach ((array) $paths as $path) {
        foreach ($this->getPossibleViewFiles($name) as $file) {
            if ($this->files->exists($viewPath = $path.'/'.$file)) {
                return $viewPath;
            }
        }
    }

    throw new InvalidArgumentException("View [{$name}] not found.");
}

/**
 * Get an array of possible view files.
 *
 * @param  string  $name
 * @return array
 */
protected function getPossibleViewFiles($name)
{
    return array_map(function ($extension) use ($name) {
        return str_replace('.', '/', $name).'.'.$extension;
    }, $this->extensions);
}
```

`getPossibleViewFiles`将view名称化成带有后缀的名称，`findInPaths`就从给定的`$paths`中，找到对应的view`$name`，而`findNamespaceView`
则是先找到命名空间下的候选路径，然后再传递给`findInPaths`。

我们还能为查找器增加候选路径，`addLocation`, `prependLocation`, `addNamespace`以及`prependNamespace`就负责这方面的工作

```php
//src/Illuminate/View/FileViewFinder.php

/**
 * Add a location to the finder.
 *
 * @param  string  $location
 * @return void
 */
public function addLocation($location)
{
    $this->paths[] = $location;
}

/**
 * Prepend a location to the finder.
 *
 * @param  string  $location
 * @return void
 */
public function prependLocation($location)
{
    array_unshift($this->paths, $location);
}

/**
 * Add a namespace hint to the finder.
 *
 * @param  string  $namespace
 * @param  string|array  $hints
 * @return void
 */
public function addNamespace($namespace, $hints)
{
    $hints = (array) $hints;

    if (isset($this->hints[$namespace])) {
        $hints = array_merge($this->hints[$namespace], $hints);
    }

    $this->hints[$namespace] = $hints;
}

/**
 * Prepend a namespace hint to the finder.
 *
 * @param  string  $namespace
 * @param  string|array  $hints
 * @return void
 */
public function prependNamespace($namespace, $hints)
{
    $hints = (array) $hints;

    if (isset($this->hints[$namespace])) {
        $hints = array_merge($hints, $this->hints[$namespace]);
    }

    $this->hints[$namespace] = $hints;
}
```