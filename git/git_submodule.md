# 

## Git 修改.Submodule文件 url 生效
* 修改 .gitmodules 文件中对应模块的url属性;
* 使用 git submodule sync 命令，将新的URL更新到文件.git/config；
* 再使用命令初始化子模块：git submodule init
* 最后使用命令更新子模块：git submodule update