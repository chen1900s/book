##  kubectl  **配置别名** 

```
# vim ~/.bashrc

添加 alias k='kubectl'

# source ~/.bashrc
```

![1628170828155](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042122318.png)

## 启用 shell 自动补全功能

 kubectl 为 Bash 和 Zsh 提供自动补全功能，可以减轻许多输入的负担。 

### 简介

kubectl 的 Bash 补全脚本可以用命令 `kubectl completion bash` 生成。 在 shell 中导入（Sourcing）补全脚本，将启用 kubectl 自动补全功能。

然而，补全脚本依赖于工具 [**bash-completion**](https://github.com/scop/bash-completion)， 所以要先安装它（可以用命令 `type _init_completion` 检查 bash-completion 是否已安装）

```
root@VM-0-17-tlinux ~]# type _init_completion
-bash: type: _init_completion: not found
#表示未安装
```

### 安装 bash-completion

apt-get install bash-completion 或 yum install bash-completion 等命令来安装它

```
 yum install bash-completion -y
 #上述命令将创建文件 /usr/share/bash-completion/bash_completion，它是 bash-completion 的主脚本
 ls /usr/share/bash-completion/bash_completion
```

依据包管理工具的实际情况，你需要在 ~/.bashrc 文件中手工导入此文件。

```
[root@VM-0-17-tlinux ~]# source /usr/share/bash-completion/bash_completion
#重新加载 shell，再输入命令 type _init_completion 来验证 bash-completion 的安装状态
[root@VM-0-17-tlinux ~]# type _init_completion
```

### 启动 kubectl 自动补全功能

 现在需要确保一点：kubectl 补全脚本已经导入（sourced）到 shell 会话中。 这里有两种验证方法： 

 在文件 `~/.bashrc` 中导入（source）补全脚本： 

```
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

 将补全脚本添加到目录 `/etc/bash_completion.d` 中： 

```
kubectl completion bash >/etc/bash_completion.d/kubectl
```

参考链接：https://kubernetes.io/zh/docs/tasks/tools/install-kubectl-linux/



最简单：

```
配置别名

# vim ~/.bashrc
添加 alias k=‘kubectl‘
# source ~/.bashrc

配置命令行补齐

# yum install -y bash-completion
# chmod +x /usr/share/bash-completion/bash_completion
# /usr/share/bash-completion/bash_completion
# source /usr/share/bash-completion/bash_completion
# source <(kubectl completion bash)
# kubectl desc<TAB> no<TAB> node<TAB>

问题：在使用命令行补齐时，必须使用kubectl的全称，使用k不行。

优化：
# source <(kubectl completion bash | sed s/kubectl/k/g)
# k desc<TAB> no<TAB> mas<TAB>
为了每次登录，都可以自动加载：
# echo "source <(kubectl completion bash)" >> /etc/bashrc

yum -y install bash-completion                           #执行安装
chmod +x /usr/share/bash-completion/bash_completion
source /usr/share/bash-completion/bash_completion        #执行加载
source <(kubectl completion bash)	                     #临时生效
echo "source <(kubectl completion bash)" >> ~/.bashrc	#当前用户永久生效

```

