本题需要构造一个ctfshow类触发__destruct魔术方法，输出flag。
通过大小写绕过正则
```
<?php
class Ctfshow{
}
$a = new Ctfshow();
echo serialize($a);
?>
```
flag : ctfshow{6479dadd-1745-491d-a222-7f46432c04a4}