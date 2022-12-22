# JD.Army
使用hexo网页生成器

## 安装hexo
npm install hexo-cli -g
或
brew install hexo

## 初始化环境
```bash
git clone https://github.com/JDArmy/JDArmy.github.io.git
cd JD.Army

rm -rf node_modules && npm install --force

#安装皮肤 && 替换静态资源
git submodule add https://github.com/fi3ework/hexo-theme-archer.git themes/archer --depth=1 
#  若提示'themes/archer' already exists in the index 时使用命令 git submodule update --init 代替

rm -rf themes/archer/_config.yml
cp -r CopyToThemes/* themes/
```

## check是否安装正确
```bash
# 本地查看效果
hexo server
```
## 写新文章
> 文章名可以中文
```bash
# 先同步到最新
git pull
# 创建新文章
npx hexo new post "article name"
# 点击链接打开md文档
# 补充tags
tags: [AD, Exchange, RCE, DACL, CVE, CobaltStrike, 蜜罐]
# 补充categories
categories: 蓝军推送
# 补充文章内容，按照一般的Markdown格式
#注意：文章中不要使用一级标题，用二级及以下标题，否则会出错

```
## 网站更新
> 注：archer模板不需要更新，更新JDArmy.github.io即可
```bash
#生成网站数据到 /docs 文件夹中
npx hexo generate
#提交到仓库
git push
```


