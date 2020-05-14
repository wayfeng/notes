# PRC network issues and solutions

# pip

Using local pip mirrors.
```sh
pip config set global.trusted-host mirrors.aliyun.com
pip config set global.index-url http://mirrors.aliyun.com/pypi/simpleyun.com
```

# apt

Using local apt mirrors.
```sh
sudo sed -i 's/cn.archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
```

# bypass DNS

```sh
$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 iotg-lab-007

xxx.xxx.xxx.xxx raw.githubusercontent.com
xxx.xxx.xxx.xxx amazonaws.com
```

# Replace github.com

```gitconfig
[url "http://gitee.com/waynefeng/onnxruntime.git"]
        insteadOf = https://github.com/microsoft/onnxruntime.git
```
