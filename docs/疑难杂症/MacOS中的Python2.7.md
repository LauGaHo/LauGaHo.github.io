# MacOS 中的 Python2.7

## 问题缘由

在拉取 `ElementUI` 的源码中，在项目中执行 `yarn` 命令时，显示 `node-sass` 这个依赖安装的时候无法获取对应的 Python2.7 的依赖，所以导致了依赖安装失败了。在依赖安装的过程中，经历了好几次放弃，最终还是那该死的胜负欲让我找到了答案。

## 问题原因

> 从 MacOS 12.3 Beta 版本开始，Apple 不再内置 Python2，并且无法正常安装 Python2，无论是 Intel 芯片还是 Apple 芯片的设备都无法安装。原因是 `usr/bin/python` 的软连接无法被正常删除或覆盖。

## 解决方式

迎来曙光：

> 在 2022/4/17 更新到：从 MacOS 12.4 Beta 版本开始，可以通过 `pyenv` 在 Intel 和 Apple 芯片中安装 Python2。

在 M 系列芯片中安装 2.7.18 版本的 Python2。

```bash
brew install pyenv

pyenv install 2.7.18

export PATH="$(pyenv root)/shims:${PATH}"

pyenv global 2.7.18

python --version
```

顺利的话，可以看到 `Python 2.7.18` 的输出。

需要将上述路径添加到环境变量中，例如：

```bash
echo 'PATH=$(pyenv root)/shims:$PATH' >> ~/.zshrc
```

此方法可以和 `brew install python3` 方式安装的 Python3 共存。
