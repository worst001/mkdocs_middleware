### 编写快速安全Bash脚本的建议
+   [编写快速安全 Bash 脚本的建议 ](https://www.oschina.net/translate/bash-scripting-quirks-safety-tips)
+ 变量赋值
    + Bash变量并不要求全部大写，但是通常是大写的
    + 变量赋值不要在`=`运算符的两边放置空格符
        + 错误写法：`VARIABLE = 2` 或者`VARIABLE =2`
        + 正确写法：`VARIABLE=2`
+ 使用${}引用变量
    + 有时某些变量，内容为file.txt，并且我想这样使用它：
        + 错误写法：`mv $MYVAR $MYVAR__bak # wrong!`
             + 这段代码是无法工作的！它会去查找MYVAR__bak变量，但这并不是一个真实存在的变量。
        + 正确使用：`mv $MYVAR ${MYVAR}__bak # right!`    
+ 全局变量，局部变量和环境变量        
            
