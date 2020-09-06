## laravel 源码解读

### 入口文件

#### 自动加载

1. 在index文件开始，首先require启动文件，这里会初始化application，返回的$app就是laravel的application

```php
$app = require __DIR__.'/../bootstrap/app.php';  
```

2. 在bootstrap/app.php文件里面，首先会加载composer的类加载器

```php
require_once __DIR__ . '/../vendor/autoload.php';
```

3. 在vendor/autoload.php这个文件里，源代码如下：

```php
require_once __DIR__ . '/composer/autoload_real.php';
return ComposerAutoloaderInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::getLoader();
```

4. 在getLoader方法里面，首先会判断是否使用静态加载器,我的php是7.0且不是HHVM，不符合if的判断，故最后会使用静态加载器

```php
$useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
if ($useStaticLoader) {
    require_once __DIR__ . '/autoload_static.php';

    call_user_func(\Composer\Autoload\ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::getInitializer($loader));
} else {
    $map = require __DIR__ . '/autoload_namespaces.php';
    foreach ($map as $namespace => $path) {
        $loader->set($namespace, $path);
    }

    $map = require __DIR__ . '/autoload_psr4.php';
    foreach ($map as $namespace => $path) {
        $loader->setPsr4($namespace, $path);
    }

    $classMap = require __DIR__ . '/autoload_classmap.php';
    if ($classMap) {
        $loader->addClassMap($classMap);
    }
}
```

5. 查看call_user_func(\Composer\Autoload\ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::getInitializer($loader))，这里主要给loader赋值，由于ClassLoader的属性都是私有的，所以使用闭包类的bind属性给loader赋值，bind的第二个参数null代表不绑定到具体的对象。赋值的都是一些类命名空间和文件路径的映射，这样就可以快速通过类的名字找到对应的类所在的文件

```php
public static function getInitializer(ClassLoader $loader)
{
    return \Closure::bind(function () use ($loader) {
        $loader->prefixLengthsPsr4 = ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$prefixLengthsPsr4;
        $loader->prefixDirsPsr4 = ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$prefixDirsPsr4;
        $loader->fallbackDirsPsr4 = ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$fallbackDirsPsr4;
        $loader->prefixesPsr0 = ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$prefixesPsr0;
        $loader->classMap = ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$classMap;

    }, null, ClassLoader::class);
}
```

6. 下一步，则是注册类的加载方法

```php
$loader->register(true);

if ($useStaticLoader) {
    $includeFiles = Composer\Autoload\ComposerStaticInitbbd0d6e1abdbbdf3f6dba38879fc3ff3::$files;
} else {
    $includeFiles = require __DIR__ . '/autoload_files.php';
}
foreach ($includeFiles as $fileIdentifier => $file) {
    composerRequirebbd0d6e1abdbbdf3f6dba38879fc3ff3($fileIdentifier, $file);
}

```

7. 查看注册方法，使用loader对象的loadClass方法来实现全部类的自动加载

```php
public function register($prepend = false)
{
    spl_autoload_register(array($this, 'loadClass'), true, $prepend);
}
```

8. 查看loadClass方法，当我们在代码中使用某个类时，php就会触发这个方法，首先findFile，找到了就include进来

```php
public function loadClass($class)
{
    if ($file = $this->findFile($class)) {
        includeFile($file);

        return true;
    }
}
```

9. 查看findFile方法，首先会在类加载器对象的classMap中进行查找，找不到会调findFileWithExtension方法。至于apcuPrefix，这应该是一个php的缓存扩展

```php
/**
 * Finds the path to the file where the class is defined.
 *
 * @param string $class The name of the class
 *
 * @return string|false The path if found, false otherwise
 */
public function findFile($class)
{
    // class map lookup
    if (isset($this->classMap[$class])) {
        return $this->classMap[$class];
    }
    if ($this->classMapAuthoritative || isset($this->missingClasses[$class])) {
        return false;
    }
    if (null !== $this->apcuPrefix) {
        $file = apcu_fetch($this->apcuPrefix.$class, $hit);
        if ($hit) {
            return $file;
        }
    }

    $file = $this->findFileWithExtension($class, '.php');

    // Search for Hack files if we are running on HHVM
    if (false === $file && defined('HHVM_VERSION')) {
        $file = $this->findFileWithExtension($class, '.hh');
    }

    if (null !== $this->apcuPrefix) {
        apcu_add($this->apcuPrefix.$class, $file);
    }

    if (false === $file) {
        // Remember that this class does not exist.
        $this->missingClasses[$class] = true;
    }

    return $file;
}
```

10. 查看findFileWithExtension方法，首先会查找符合psr-4规范的类，找不到会查找符合psr-0规范的类，故类的自动加载是这样实现的

```php
private function findFileWithExtension($class, $ext)
{
    // PSR-4 lookup
    $logicalPathPsr4 = strtr($class, '\\', DIRECTORY_SEPARATOR) . $ext;

    $first = $class[0];
    if (isset($this->prefixLengthsPsr4[$first])) {
        $subPath = $class;
        while (false !== $lastPos = strrpos($subPath, '\\')) {
            $subPath = substr($subPath, 0, $lastPos);
            $search = $subPath.'\\';
            if (isset($this->prefixDirsPsr4[$search])) {
                $pathEnd = DIRECTORY_SEPARATOR . substr($logicalPathPsr4, $lastPos + 1);
                foreach ($this->prefixDirsPsr4[$search] as $dir) {
                    if (file_exists($file = $dir . $pathEnd)) {
                        return $file;
                    }
                }
            }
        }
    }

    // PSR-4 fallback dirs
    foreach ($this->fallbackDirsPsr4 as $dir) {
        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr4)) {
            return $file;
        }
    }

    // PSR-0 lookup
    if (false !== $pos = strrpos($class, '\\')) {
        // namespaced class name
        $logicalPathPsr0 = substr($logicalPathPsr4, 0, $pos + 1)
            . strtr(substr($logicalPathPsr4, $pos + 1), '_', DIRECTORY_SEPARATOR);
    } else {
        // PEAR-like class name
        $logicalPathPsr0 = strtr($class, '_', DIRECTORY_SEPARATOR) . $ext;
    }

    if (isset($this->prefixesPsr0[$first])) {
        foreach ($this->prefixesPsr0[$first] as $prefix => $dirs) {
            if (0 === strpos($class, $prefix)) {
                foreach ($dirs as $dir) {
                    if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
                        return $file;
                    }
                }
            }
        }
    }

    // PSR-0 fallback dirs
    foreach ($this->fallbackDirsPsr0 as $dir) {
        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
            return $file;
        }
    }

    // PSR-0 include paths.
    if ($this->useIncludePath && $file = stream_resolve_include_path($logicalPathPsr0)) {
        return $file;
    }

    return false;
}
```

