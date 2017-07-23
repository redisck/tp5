经过精简的第三方登录代码，比腾讯官方示例还简单。
使用方法：
1、将控制器类拷贝到应用目录下，如home目录；
2、在config.php文件中配置登录账号信息：
//第三方登录
'thirdlogin'	=>[
'qq' => [
'appid'=> '',
'appsecret'=>'',
],
'weixin'=>[
'appid'=>'',
'appsecret'=>'',
]
],
3、在view类的页面文件中，写入url。
<li><a href="{:url('Qqlogin/index')}">QQ</a></li>
<li><a href="{:url('Wxlogin/index')}">微信</a></li>

注：对于公司来说，微信必须是经过认证的服务号，QQ需要通过审核的域名
