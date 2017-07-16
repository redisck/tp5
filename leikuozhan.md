关于thinkphp5.0 rc4.0扩展类库自动与手动加载的问题
《ThinkPHP5.0完全开发手册》一共提供了三种方式实现扩展类库加载。
下面，我将这三种方式根据实测案例道来：
第一种：自动注册
把类库包放到EXTEND_PATH目录（extend，可配置），就可以自动注册对应的命名空间。
其原理是：tp5遵循了PSR-4自动加载规范，只要类的文件名均以命名空间定义，并且命名空间的路径和类库文件所在路径一致，便可实现自动加载。
（假设是框架默认目录结构）执行public下的入口文件index.PHP，会执行到 注册自动加载
\think\Loader::register();而该方法最后会执行自动加载extend目录self::$fallbackDirsPsr4[] = rtrim(EXTEND_PATH, DS);这个EXTEND_PATH常量是在base.php预先定义好的，defined('EXTEND_PATH') or define('EXTEND_PATH', ROOT_PATH . 'extend' . DS);其相对于入口文件的路径为：../extend/ ,故我们在extend目录下面新增一个my目录，然后定义一个\my\Test类（ 类文件位于extend/my/Test.php），Test.php文件内容如下：
namespace my;

class Test 
{
    public function sayHello()
    {
        return 'hello';
    }
}
当你在控制器中直接调用时，如：var_dump((new \my\Test())->sayHello());输出结果为：string(5) "hello"。
因此，当我们要变动类库文件目录结构时，例如我们想让my文件夹还有父级目录vendor，只需在入口文件将define('EXTEND_PATH','../extend/vendor/');放到加载框架引导文件require __DIR__ . '/../thinkphp/start.php';之前，然后将Test.php文件目录结构变成extend/vendor/my/Test.php就ok了。
第一种方法十分简单，个人推荐使用这种方式。因此我把它放在第一个介绍。


第二种方法：修改应用配置文件实现扩展类库自动加载
原理：这种方式是建立命名空间和类库文件路径的映射关系。
修改应用配置文件（默认框架目录结构application/config.php），注意不是惯例管理配置文件thinkphp/convention.php（tp5优先加载惯例配置，后加载应用配置，后加载的配置会覆盖先加载的）。
还是上面那个例子，我们先注释掉入口文件这一行define('EXTEND_PATH','../extend/vendor/');然后运行，你会发现报错信息提示为：Class
 'my\Test' not found.
我们修改application/config.php，修改
 注册的根命名空间 'root_namespace'=> ['my'  => '../extend/vendor/my/',],然后保存在执行！又出现hello了。
这种方式是建立命名空间和类库文件路径的映射关系。


第三种方式：手动注册新的根命名空间
原理：同第二种方法(用例还是extend/vendor/my/Test.php)
但《ThinkPHP5.0完全开发手册》给出的样例绝对不好用的。如下图：
在应用入口文件中添加下面的代码：
\think\Loader::addNamespace('my','../application/extend/my/');
当你把上面的配置文件'root_namespace'=>
 ['my'  => '../extend/vendor/my/',],注释掉时，运行肯定报错。其一是默认的框架目录结构extend和application是同一级目录。其二,my的父目录不是extend,而是vendor。
好，你说不对，那我改成
\think\Loader::addNamespace('my', '../extend/vendor/my/',);
再运行，怎么还是Class 'my\Test' not found。
这里，修改方案有两种。
第一解决办法，你可以将
\think\Loader::addNamespace('my', '../extend/vendor/my/');
放在thinkphp/start.php文件内容的中间位置，修改如下：
namespace think;
// ThinkPHP 引导文件
// 加载基础文件
require __DIR__ . '/base.php';

#########中间#########
\think\Loader::addNamespace('my', '../extend/vendor/my/');

// 执行应用
\think\App::run()->send();

第二解决办法，类似TP3.2.2的入口文件index.php，注释掉thinkphp/start.php文件中的\think\Loader::addNamespace('my', '../extend/vendor/my/');   将\think\App::run()->send();移到index.php文件尾部。修改如下：
define('APP_PATH', __DIR__ . '/../application/');
// 加载框架引导文件
require __DIR__ . '/../thinkphp/start.php';

########################增加的代码
\think\Loader::addNamespace('my', '../extend/vendor/my/');
// 执行应用
\think\App::run()->send();
运行看看，是不是hello有出来了？偷笑
发现猫腻没有？应先注册根命名空间my，再来执行应用，这样执行到控制器那里就会根据my命名空间对应的路径找到Test.php了。
官方手册给出错误的例子，实际上是让我们能深入理解整个框架的执行过程！
发现错误，找出错误原因，涨知识，活运用！让我们做一个快乐的程序猿！
