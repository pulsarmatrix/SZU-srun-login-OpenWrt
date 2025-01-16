# 概述

深圳大学深澜校园网登录python脚本，可用于任何支持python的设备的网络命令行登录或命令行登录。

从北理工的版本修改而来，特别鸣谢 [北理工深澜登录脚本](https://github.com/coffeehat/BIT-srun-login-script)


# 文件说明

|      文件       |     说明      |
|:-------------:|:-----------:|
|    main.py    |     主程序     |
|    apis.py    |   登录请求脚本    |
| decorators.py | 用于校验和提示的装饰器 |
|  encrypt.py   |    登录加密用    |

main.py可采用`nohup`命令挂在后台：
``` bash
nohup python main.py &
```

简单写一下，希望能帮到大家🤪
