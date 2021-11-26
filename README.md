# JD.Army
使用hexo网页生成器
## 初始化环境
```bash
git clone https://github.com/JDArmy/JDArmy.github.io.git
cd JD.Army
npm install
#安装皮肤 && 替换静态资源
git submodule add https://github.com/fi3ework/hexo-theme-archer.git themes/archer --depth=1
rm -rf themes/archer/_config.yml
cp -r CopyToThemes/* themes/
```
## check是否安装正确
```bash
# 本地查看效果
hexo s
```
## 写新文章
> 文章名可以中文
```bash
git pull
hexo new post "article name"
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
hexo g
#提交到仓库
git push
```