 使用thinkphp如何加载自定义类
 在研究thinkphp框架时，才真正的了解到thinkphp是单入口文件。通过项目下的index.PHP进入项目程序，然后进入ThinkPHP公共入口文件，公共文件主要定义一些路径常量，方便以后程序的使用，如图：

然后就进入了比较重要的含金量大大的ThinkPHP引导类。本博客中有相关的文章对该类进行了详细的解读，在此不再详细的叙说。
　　在该类的start方法中有这样一段代码：
// 加载模式别名定义
if(isset($mode['alias'])){
    self::addMap(is_array($mode['alias'])?$mode['alias']:include $mode['alias']);
}
 
// 加载应用别名定义文件
if(is_file(CONF_PATH.'alias.php'))
self::addMap(include CONF_PATH.'alias.php');　
  　这段代码是加载系统定义的基础类库和扩展类库的。定义的文件位置分别为./application/common/conf/alias.php（定义扩展类库） 和。./ThinkPHP/Mode/common.php

　　没有可以自己定义，显然这个目录下就没有，可以自己定义。

文件部分内容如下：

都是系统基础类库。
　　比如我有一个类文件叫做：Lunar.class.php将其放在/Think/目录下如图：

最后一个就是我的类，然后在上述的common.php中添加一行
　　'Think\\Lunar' => CORE_PATH . 'Lunar'.EXT,
　　最后再你的控制器中就可以使用这个类了
$lunar = new \\Think\\Lunar();
$ldate=$lunar->convertSolarToLunar(date("Y"), date("m"), date("d"));
$smonth=date("m",time());
 
