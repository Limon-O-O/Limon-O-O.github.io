---
layout: post
title: "CheatSheet"
date: 2015-12-23 02:22:06 +0800
comments: true
categories: CheatSheet
---

判断一个view是否是另一个view的子视图
```
- (BOOL)isDescendantOfView:(UIView *)view;
```

隐藏或显示"隐藏文件"
```
defaults write com.apple.finder AppleShowAllFiles -boolean true; killall Finder
```

### Octopress
```
rake "new_post[Post Title]" // zsh下
rake generate                                  生成html文件
rake preview
rake deploy                                    部署文章（博客）
```
```
git add .
git commit -m 'initial source commit'
git push origin source
```