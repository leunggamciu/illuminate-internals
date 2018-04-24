# 文件引擎

文件引擎的实现很简单，就是读取给定的模板文件，然后返回读取到的内容。它没法处理数据。


```php
//src/Illuminate/View/Engines/FileEngine.php


class FileEngine implements Engine
{
    /**
     * Get the evaluated contents of the view.
     *
     * @param  string  $path
     * @param  array   $data
     * @return string
     */
    public function get($path, array $data = [])
    {
        return file_get_contents($path);
    }
}
```