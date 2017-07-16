Thinkphp5类加载机制
PS：本文适用于熟悉Thinkphp框架或其他MVC框架，对于命名空间及自动加载类，有一定理解的程序工作者观看(其实是写给自己看的>_<—予怀之言)
我一直对于thinkphp5的加载运行的时候做了什么，他是怎么自动加载类的，是和composer一样的吗—能否实现运行时再加载类，于是做了一下测试，追踪了整个加载流程。
 
以文件为单位进行讲解：
入口文件index.PHP
<?php
// 定义应用目录
define('APP_PATH', __DIR__ . '/../application/');
// 加载框架引导文件
require __DIR__ . '/../thinkphp/start.php';
 
1. 定义了APP_PATH常量
2. 加载start.php文件
 
start.php文件
<?php
namespace think;

// ThinkPHP 引导文件
// 加载基础文件
require __DIR__ . '/base.php';
// 执行应用
App::run()->send();
 
1. 加载base.php
2. 执行App类的静态方法run(),再执行返回值的send()方法
来到这里，出现两个问题，App类。。。是从哪里加载进来的呢？看看base.php
 
base.php文件
<?php
define('THINK_VERSION', '5.0.3');
define('THINK_START_TIME', microtime(true));
define('THINK_START_MEM', memory_get_usage());
define('EXT', '.php');
define('DS', DIRECTORY_SEPARATOR);
defined('THINK_PATH') or define('THINK_PATH', __DIR__ . DS);
define('LIB_PATH', THINK_PATH . 'library' . DS);
define('CORE_PATH', LIB_PATH . 'think' . DS);
define('TRAIT_PATH', LIB_PATH . 'traits' . DS);
defined('APP_PATH') or define('APP_PATH', dirname($_SERVER['SCRIPT_FILENAME']) . DS);
defined('ROOT_PATH') or define('ROOT_PATH', dirname(realpath(APP_PATH)) . DS);
defined('EXTEND_PATH') or define('EXTEND_PATH', ROOT_PATH . 'extend' . DS);
defined('VENDOR_PATH') or define('VENDOR_PATH', ROOT_PATH . 'vendor' . DS);
defined('RUNTIME_PATH') or define('RUNTIME_PATH', ROOT_PATH . 'runtime' . DS);
defined('LOG_PATH') or define('LOG_PATH', RUNTIME_PATH . 'log' . DS);
defined('CACHE_PATH') or define('CACHE_PATH', RUNTIME_PATH . 'cache' . DS);
defined('TEMP_PATH') or define('TEMP_PATH', RUNTIME_PATH . 'temp' . DS);
defined('CONF_PATH') or define('CONF_PATH', APP_PATH); // 配置文件目录
defined('CONF_EXT') or define('CONF_EXT', EXT); // 配置文件后缀
defined('ENV_PREFIX') or define('ENV_PREFIX', 'PHP_'); // 环境变量的配置前缀

// 环境常量
define('IS_CLI', PHP_SAPI == 'cli' ? true : false);
define('IS_WIN', strpos(PHP_OS, 'WIN') !== false);

// 载入Loader类
require CORE_PATH . 'Loader.php';

// 加载环境变量配置文件
if (is_file(ROOT_PATH . '.env')) {
    $env = parse_ini_file(ROOT_PATH . '.env', true);
    foreach ($env as $key => $val) {
        $name = ENV_PREFIX . strtoupper($key);
        if (is_array($val)) {
            foreach ($val as $k => $v) {
                $item = $name . '_' . strtoupper($k);
                putenv("$item=$v");
            }
        } else {
            putenv("$name=$val");
        }
    }
}

// 注册自动加载
\think\Loader::register();

// 注册错误和异常处理机制
\think\Error::register();

// 加载惯例配置文件
\think\Config::set(include THINK_PATH . 'convention' . EXT);
 
1. 定义了许多常量
2. 载入Loader类
3. 查看有无“环境变量配置文件”，有的话就加载
4. 运行Loader类的register()方法
5. 运行Error类的register()方法
 
从上下文以及注释可以知道，应该是Loader类的静态方法register()方法，实现了类的自动加载。重点就是它了。
 
Loader类的静态方法register()方法
// 注册自动加载机制
public static function register($autoload = '')
{
    // 注册系统自动加载
    spl_autoload_register($autoload ?: 'think\\Loader::autoload', true, true);
    // 注册命名空间定义
    self::addNamespace([
        'think'    => LIB_PATH . 'think' . DS,
        'behavior' => LIB_PATH . 'behavior' . DS,
        'traits'   => LIB_PATH . 'traits' . DS,
    ]);
    // 加载类库映射文件
    if (is_file(RUNTIME_PATH . 'classmap' . EXT)) {
        self::addClassMap(__include_file(RUNTIME_PATH . 'classmap' . EXT));
    }

    // Composer自动加载支持
    if (is_dir(VENDOR_PATH . 'composer')) {
        self::registerComposerLoader();
    }

    // 自动加载extend目录
    self::$fallbackDirsPsr4[] = rtrim(EXTEND_PATH, DS);
}
 
1. 默认$autoload为空，使用PHP内置函数spl_autoload_register注册Loader类的静态方法autoload()方法为自动加载机制。
其意思即是，当使用类的时候，找不到类或者还没有包含，会将类名传给autoload()方法，让它处理
2. 注册命名空间定义，是将命名空间与真实的文件路径对应一一对应，并且存储在一个类静态变量数组里面。
其作用是，在autoload()方法接收到类的名字的时候，通过类所属的命名空间，找到类文件的真实路径，进而将类文件包含include
3. 以下的基本是一样的加载，只不过不是加载框架自身的类文件，而是加载composer库或者自己写的类
 
autoload()方法
// 自动加载
public static function autoload($class)
{
    // 检测命名空间别名
    if (!empty(self::$namespaceAlias)) {
        $namespace = dirname($class);
        if (isset(self::$namespaceAlias[$namespace])) {
            $original = self::$namespaceAlias[$namespace] . '\\' . basename($class);
            if (class_exists($original)) {
                return class_alias($original, $class, false);
            }
        }
    }

    if ($file = self::findFile($class)) {

        // Win环境严格区分大小写
        if (IS_WIN && pathinfo($file, PATHINFO_FILENAME) != pathinfo(realpath($file), PATHINFO_FILENAME)) {
            return false;
        }

        __include_file($file);
        return true;
    }
}
 
如你所见，autoload方法就是接收类名，搜索类文件，包含类文件。
 
恩，至此Thinkphp5运行时加载类的流程已经明了。
总结一下，index.php加载start.php,在start.php里面加载base.php(类自动加载机制就在这里出现了)，然后下面调用App的静态方法run方法执行“模块/控制器/操作”，返回Respose类的实例执行send方法，将响应数据发送给客户端，这样，一个完整的请求就完成了。 

附录：
各个文件的路径：
index.php
\think5\public
 
start.php
\think5\thinkphp
 
base.php
\think5\thinkphp
 
Loader.php
\think5\thinkphp\library\think
 
对于spl_autoload_register与自动加载有疑惑或不懂的可以看看我的另外一篇博客
__autoload,spl_autoload_register与自动加载
 
