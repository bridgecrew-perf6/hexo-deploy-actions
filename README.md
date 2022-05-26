<p align="center">
  <a href="https://github.com/actions/typescript-action/actions"><img alt="typescript-action status" src="https://github.com/actions/typescript-action/workflows/build-test/badge.svg"></a>
</p>

# Hexo Deploy Sync Gitee

🔄 Automatically deploy Hexo to Github Pages and synchronize the Gitee repository

示例代码:

```yml
name: Hexo Deploy Sync Gitee

on:
  push:
    branches:
      - main

# Action 的名字
name: Hexo Auto Deploy

on:
  # 触发条件1：main 分支收到 push 后执行任务。
  push:
    branches:
      - main
  # 触发条件2：手动按钮
  workflow_dispatch:

# 这里放环境变量,需要替换成你自己的
env:
  # Hexo 编译后使用此 git 用户部署到 github 仓库
  GIT_USER: wbsu2003
  # Hexo 编译后使用此 git 邮箱部署到 github 仓库
  GIT_EMAIL: wbsu2003@gmail.com
  # Hexo 编译后要部署的 github 仓库
  GIT_DEPLOY_REPO: wbsu2003/wbsu2003.github.io
  # Hexo 编译后要部署到的分支
  GIT_DEPLOY_BRANCH: master
  
  # Hexo 编译后使用此 gitee 用户部署到gitee仓库
  GITEE_USER: wbsu2003
  # Hexo 编译后要部署的 gitee 仓库
  GITEE_DEPLOY_REPO: wbsu2003/wbsu2003
  # Hexo 编译后要部署到的分支
  GITEE_DEPLOY_BRANCH: master
  
  # 注意替换为你的 GitHub 源仓库地址
  GIT_SOURCE_REPO: git@github.com:wbsu2003/wbsu2003.github.io.git
  # 注意替换为你的 Gitee 目标仓库地址
  GITEE_DESTINATION_REPO: git@gitee.com:wbsu2003/wbsu2003.git

jobs:
  build:
    name: Build on node ${{ matrix.node_version }} and ${{ matrix.os }}
    runs-on: ubuntu-latest
    if: github.event.repository.owner.id == github.event.sender.id
    strategy:
      matrix:
        os: [ubuntu-18.04]
        node_version: [12.x]

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Checkout deploy repo
        uses: actions/checkout@v2
        with:
          repository: ${{ env.GIT_DEPLOY_REPO }}
          ref: ${{ env.GIT_DEPLOY_BRANCH }}
          path: .deploy_git

      - name: Use Node.js ${{ matrix.node_version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}

      - name: Configuration environment
        env:
          HEXO_DEPLOY_PRI: ${{secrets.HEXO_DEPLOY_PRI}}
        run: |
          sudo timedatectl set-timezone "Asia/Shanghai"
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRI" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts
          # coding 已取消同步
          ssh-keyscan -t rsa e.coding.net >> ~/.ssh/known_hosts
          ssh-keyscan -t rsa gitee.com >> ~/.ssh/known_hosts
          git config --global user.name $GIT_USER
          git config --global user.email $GIT_EMAIL

      - name: Install dependencies
        run: |
          npm install hexo-cli -g
          npm install
		  # 根据你安装的组件进行安装
          npm uninstall hexo-generator-index --save
          npm install hexo-baidu-url-submit hexo-generator-index2 hexo-symbols-count-time hexo-blog-encrypt hexo-deployer-git --save
          # 复制中文语言包，解决菜单英文的问题
          cp zh-CN.yml node_modules/hexo-theme-next/languages/

      - name: Deploy hexo
        run: |
          npm run deploy
		  
      # 以下为发布到gitee
      - name: Sync to Gitee
        uses: wearerequired/git-mirror-action@master
        env:
          # 直接使用了 HEXO_DEPLOY_PRI
          SSH_PRIVATE_KEY: ${{ secrets.HEXO_DEPLOY_PRI }}
        with:
          # GitHub 源仓库地址
          source-repo: ${{ env.GIT_SOURCE_REPO }}
          # Gitee 目标仓库地址
          destination-repo: ${{ env.GITEE_DESTINATION_REPO }}

      - name: Build Gitee Pages
        uses: yanglbme/gitee-pages-action@main
        with:
          # 你的 Gitee 用户名
          gitee-username: ${{ env.GITEE_USER }}
          # 注意在 Settings->Secrets 配置 GITEE_PASSWORD
          gitee-password: ${{ secrets.GITEE_PASSWORD }}
          # 你的 Gitee 仓库，仓库名严格区分大小写，请准确填写，否则会出错
          gitee-repo: ${{ env.GITEE_DEPLOY_REPO }}
          # 要部署的分支，默认是 master，若是其他分支，则需要指定（指定的分支必须存在）
          branch: ${{ env.GITEE_DEPLOY_BRANCH }}

```