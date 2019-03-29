# git使用
>git push --help

## .gitignore
### git忽略规则
```
#此为注释 – 内容被 Git 忽略
.sample 　　 # 忽略所有 .sample 结尾的文件
!lib.sample 　　 # 但 lib.sample 除外
/TODO 　　 # 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
build/ 　　 # 忽略 build/ 目录下的所有文件
doc/.txt 　　# 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
```
### .gitignore规则不生效的解决办法
把某些目录或文件加入忽略规则，按照上述方法定义后发现并未生效，原因是.gitignore只能忽略那些原来没有被追踪的文件，如果某些文件已经被纳入了版本管理中，则修改.gitignore是无效的。那么解决方法就是先把本地缓存删除（改变成未被追踪状态），然后再提交：
```shell
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```