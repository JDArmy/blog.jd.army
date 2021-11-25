# JD.Army
https://JD.Army 网页生成器

## 初始化
```bash
npm install hexo -g
git clone https://github.com/JDArmy/JD.Army.git
cd JD.Army
npm install
git clone https://github.com/fi3ework/hexo-theme-archer.git themes/archer --depth=1
rm -rf themes/archer/_config.yml
cp -r CopyToThemes/* themes/
```
## check是否安装正确
```bash
# 本地查看效果
hexo s
```

## 写新文章
>> 文章名可以中文
```bash
hexo new post "article name"

# 补充tags
tags: [AD, Exchange, RCE, DACL, CVE, CobaltStrike, 蜜罐]

# 补充categories
categories: 蓝军推送

# 补充文章内容，注意：文章中不要用一级标题，用二级以下标题
```

## 网站更新
```bash
#删除旧的网站文件夹
hexo clean
#生成 /public 文件夹
hexo g 
#提交到仓库
git pull && git push
```