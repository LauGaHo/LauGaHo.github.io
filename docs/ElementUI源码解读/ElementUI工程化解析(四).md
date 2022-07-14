# ElementUI工程化解析(四)

## 发布部署

执行命令 `npm run pub` 实现发布部署流程：

- 本地代码检查合并，`push` 到远程分支。
- 组件构建发布 `npm publish`。
- 官网发布部署。

### `build/git-release.sh`

检查本地代码 `dev` 分支是否和线上分支存在冲突。

```shell
#!/usr/bin/env sh
# 切换到 dev 分支
git checkout dev

# 检查本地和暂存区是否存在未提交的文件
if test -n "$(git status --porcelain)"; then
  echo 'Unclean working tree. Commit or stash changes first.' >&2;
  exit 128;
fi

# 检查本地分支是否有错误
if ! git fetch --quiet 2>/dev/null; then
  echo 'There was a problem fetching your branch. Run `git fetch` to see more...' >&2;
  exit 128;
fi

# 检查本地分支是否落后远程分支
if test "0" != "$(git rev-list --count --left-only @'{u}'...HEAD)"; then
  echo 'Remote history differ. Please pull changes.' >&2;
  exit 128;
fi

# 通过以上检查，代码无冲突
echo 'No conflicts.' >&2;
```

### `build/release.sh`

主要作用：代码分支合并、`push` 到远程分支、版本号确认更新、组件主题发布 `npm publish`。

- 合并 dev 分支到 master 分支。
- 通过 `select-version-cli` 确认发布版本号。
- 执行命令 `npm run dist` 打包构建组件。
- 运行 ssr 测试 `node test/ssr/require.test.js`。
- 发布主题，更新版本号，和组件库保持一致。
- 提交代码并更新 `package.json` 中的版本号。
- master 和 dev 分支 `push` 到远程分支。
- 发布组件。

```shell
#!/usr/bin/env sh
# -e 当命令发生错误的时候，停止脚本的执行
set -e

# 合并 dev 分支到 master
git checkout master
git merge dev

# 安装 cli，用于交互获取更新版本信息
VERSION=`npx select-version-cli`

# 确认发布版本信息
read -p "Releasing $VERSION - are you sure? (y/n)" -n 1 -r
echo    # (optional) move to a new line
if [[ $REPLY =~ ^[Yy]$ ]]
then
  echo "Releasing $VERSION ..."

  # build 组件构建打包
  VERSION=$VERSION npm run dist

  # ssr test 测试
  node test/ssr/require.test.js            

  # publish theme
  echo "Releasing theme-chalk $VERSION ..."
  cd packages/theme-chalk
  # 更改主题包的版本信息
  npm version $VERSION --message "[release] $VERSION"
  # 发布主题
  if [[ $VERSION =~ "beta" ]]
  then
    npm publish --tag beta
  else
    npm publish
  fi
  # 回到根目录
  cd ../..

  # commit
  git add -A
  git commit -m "[build] $VERSION"
  # 更改组件库的版本信息
  npm version $VERSION --message "[release] $VERSION"

  # publish 将 master 推到远程仓库
  git push eleme master
  git push eleme refs/tags/v$VERSION
  git checkout dev
  git rebase master
  git push eleme dev

  # 发布组件库
  if [[ $VERSION =~ "beta" ]]
  then
    npm publish --tag beta
  else
    npm publish
  fi
fi
```

### `build/deploy-faas.sh`

网站发布部署，用于 `faas deploy` 配置，`2.15` 版本之后移除了 `pub` 命令 `sh build/deploy-faas.sh` 调用，集成到了 CI，详见 `build/deploy-ci.sh`。

```shell
#! /bin/sh
# -e 当命令发生错误的时候，停止脚本执行
# -x 运行的命令用一个 + 标记之后显示出来
set -ex
# 新建 temp_web 目录
mkdir temp_web
# 网站构建
npm run deploy:build
# 进入 temp_web
cd temp_web
# 拷贝 gh-pages 分支仓库到本地，进入该项目中的 temp_web/element
git clone --depth 1 -b gh-pages --single-branch https://github.com/ElemeFE/element.git && cd element

# build sub folder 创建子目录
SUB_FOLDER='2.15'
# 创建子目录 temp_web/element/2.15
mkdir -p $SUB_FOLDER
# 清除文件 temp_web/element 静态文件
rm -rf *.js *.css *.map static
# 清除文件夹 temp_web/element/2.15
rm -rf $SUB_FOLDER/**

# examples/element-ui 目录下网站静态文件 copy 到 temp_web/element 目录下
cp -rf ../../examples/element-ui/** .
# examples/element-ui 目录下网站静态文件 copy 到 temp_web/element/2.15 目录下进行归档
cp -rf ../../examples/element-ui/** $SUB_FOLDER/
# 返回根目录
cd ../..

# deploy domestic site 发布
faas deploy daily -P element
# 清除临时目录
rm -rf temp_web
```

## 持续集成

持续集成 `CI` 指的是只要代码变更了，就自动进行构建和测试，反馈运行结果。确保符合预期之后，再将新代码集成到 Master 分支，持续集成的好处是，每次代码的小幅变动，就能看到运行结果，从而不断累积小的变更，而不是在开发周期结束的时候，一下子合并一大块代码。

### `.travis.yml`

使用 Travis CI 提供的持续集成服务，项目的根目录下边必须有一个 `.travis.yml` 文件。这是一个配置文件，指定了 Travis 的行为。该文件必须保存在 GitHub 仓库中，一旦代码仓库有新的 Commit，Travis 就会去找该文件，执行里面的命令 (执行测试、构建、部署等操作)。

```yaml
# 是否需要 sudo 权限
sudo: false
# 指定默认运行环境
language: node_js
# NodeJS 版本
node_js: 10
# 插件
addons:
  chrome: stable
# install 阶段之前执行
before_install:
- export TRAVIS_COMMIT_MSG="[deploy] $(git log --format='%h - %B' --no-merges -n 1)"
- export TRAVIS_COMMIT_USER="$(git log --no-merges -n 1 --format=%an)"
- export TRAVIS_COMMIT_EMAIL="$(git log --no-merges -n 1 --format=%ae)"
# script 阶段成功时执行
after_success:
# 执行脚本 build test deploy
- sh build/deploy-ci.sh
# coveralls 自动测试代码覆盖率
- cat ./test/unit/coverage/lcov.info | ./node_modules/.bin/coveralls
```

### `build/deploy-ci.sh`

主要用于构建发行版本和开发版本内容。

- `git config` 定义 `user.name` 和 `user.email`。
- 发行版本构建 (组件库、主题 `theme-chalk`、项目网站)、打新标签。
- 开发分支的主题、项目网站构建提交到 Master 分支。

```shell
#! /bin/sh
# 创建 temp_web 文件夹
mkdir temp_web
# 配置 name email
git config --global user.name "element-bot"
git config --global user.email "wallement@gmail.com"

if [ "$ROT_TOKEN" = "" ]; then
  echo "Bye~"
  exit 0
fi

# release 发行版本
if [ "$TRAVIS_TAG" ]; then
  # build lib 组件库构建
  npm run dist
  cd temp_web
  git clone https://$ROT_TOKEN@github.com/ElementUI/lib.git && cd lib
  rm -rf `find * ! -name README.md`
  cp -rf ../../lib/** .
  git add -A .
  git commit -m "[build] $TRAVIS_TAG"
  git tag $TRAVIS_TAG
  git push origin master --tags
  cd ../..

  # build theme-chalk 主题 theme-chalk 构建
  cd temp_web
  git clone https://$ROT_TOKEN@github.com/ElementUI/theme-chalk.git && cd theme-chalk
  rm -rf *
  cp -rf ../../packages/theme-chalk/** .
  git add -A .
  git commit -m "[build] $TRAVIS_TAG"
  git tag $TRAVIS_TAG
  git push origin master --tags
  cd ../..

  # build site 项目网站构建，参考 deploy-faas.sh
  npm run deploy:build
  cd temp_web
  git clone --depth 1 -b gh-pages --single-branch https://$ROT_TOKEN@github.com/ElemeFE/element.git && cd element
  # build sub folder
  echo $TRAVIS_TAG

  SUB_FOLDER='2.15'
  mkdir $SUB_FOLDER
  rm -rf *.js *.css *.map static
  rm -rf $SUB_FOLDER/**
  cp -rf ../../examples/element-ui/** .
  cp -rf ../../examples/element-ui/** $SUB_FOLDER/
  git add -A .
  git commit -m "$TRAVIS_COMMIT_MSG"
  git push origin gh-pages
  cd ../..

  echo "DONE, Bye~"
  exit 0
fi

# build dev site 开发环境，网站构建
npm run build:file && CI_ENV=/dev/$TRAVIS_BRANCH/ node_modules/.bin/cross-env NODE_ENV=production node_modules/.bin/webpack --config build/webpack.demo.js
cd temp_web
git clone https://$ROT_TOKEN@github.com/ElementUI/dev.git && cd dev
mkdir $TRAVIS_BRANCH
rm -rf $TRAVIS_BRANCH/**
cp -rf ../../examples/element-ui/** $TRAVIS_BRANCH/
git add -A .
git commit -m "$TRAVIS_COMMIT_MSG"
git push origin master
cd ../..

# push dev theme-chalk 开发环境，主题构建
cd temp_web
git clone -b $TRAVIS_BRANCH https://$ROT_TOKEN@github.com/ElementUI/theme-chalk.git && cd theme-chalk
rm -rf *
cp -rf ../../packages/theme-chalk/** .
git add -A .
git commit -m "$TRAVIS_COMMIT_MSG"
git push origin $TRAVIS_BRANCH
cd ../..
```

## `makefile`

`makefile` 的好处就是自动化构建，一旦写好，只需要一个 `make` 命令，整个工程完全自动化，极大提高了软件的开发效率。

> `make` 是一个工具程序，经过 `make` 读取的文件就叫做 `makefile` 文件，自动化构建软件。是一种转化文件形式的工具，转换的目标称为 `target`；与此同时，它也检查文件的依赖关系，如果需要，会调用一些外部软件来完成任务。它使用名为 `makefile` 的文件来确定一个 `target` 文件的依赖关系，然后把生成这个 `target` 的相关命令传给 `shell` 去执行。

Mac 和 Linux 可以直接执行 `make` 命令，Windows 需要下载对应的工具。