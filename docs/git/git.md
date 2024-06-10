<!-- omit from toc -->
# Git

- [指令](#指令)
  - [remote](#remote)
- [徽章](#徽章)
- [gitignore文件不生效](#gitignore文件不生效)

## 指令

### remote

- 设置远程仓库地址
```bash
git remote set-url origin <NEW_URL>
```

- 查询远程仓库
```bash
git remote -v
```

## 徽章

制作网站 [shields](https://shields.io/)

## gitignore文件不生效

`.gitignore`文件只会在第一次提交项目的时候写入缓存，也就是说如果你第一次提交项目时候忘记写`.gitignore`文件，后来再补上是没有用的。

解决办法：使用`git rm -r --cached .`清除所有的缓存，然后再次提交代码就可以了。

```bash
# 清除缓存文件
git rm -r --cached .
# 再次提交文件时可以看到.gitignore生效了
git add .
git commit -m <COMMIT_MSG>
git push
```