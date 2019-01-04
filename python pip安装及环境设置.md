# python pip安装及环境设置

@(开发)[python]

# 安装pip
## 下载pip安装包

https://pypi.python.org/pypi/pip#downloads

```python
python setup.py install
```

## 设置环境变量

将 python script目录（如 D:\Python27\Scripts） 添加到环境变量中
命令行执行 pip 确认是否安装成功

# 配置工作环境
## 设置代理和配置信息
```bash
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple
extra-index-url=http://pypi.douban.com/simple
proxy=http://web-proxy.tencent.com:8080
[install]
trusted-host=
  pypi.tuna.tsinghua.edu.cn
  pypi.douban.com
```
>mac 目录： ~/.pip/pip.conf
>windows目录： C:\Users\username\pip\pip.ini   （就在C盘下，不是python安装目录）
 

