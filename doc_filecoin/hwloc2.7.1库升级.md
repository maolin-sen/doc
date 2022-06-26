# hwloc2.7.1库升级

## 安装hwloc

### 编译环境
```sh
sudo apt remove libhwloc-dev
sudo apt install autoconf
sudo apt install libtool-bin

Note that GNU autoconf >=2.69, automake >=1.16 and Libtool >=2.4.6 are required when building from a Git clone.
autoconf -V
automake --version
libtool --version

注意automake在apt库中版本较低，需要下载(wget https://mirrors.sjtug.sjtu.edu.cn/gnu/automake/automake-1.16.tar.gz)编译

```

### 编译安装
```sh
wget https://download.open-mpi.org/release/hwloc/v2.7/hwloc-2.7.1.tar.gz
tar xvzf hwloc-2.7.1.tar.gz

cd hwloc-2.7.1

./configure
make
sudo make install
```

### 库依赖链接

```sh
ln -s /usr/local/lib/libhwloc.so.15.5.3 /usr/lib/x86_64-linux-gnu/libhwloc.so.15
```

### 验证

执行一下命令
```sh
ldd hwloc-ls
```
看链接的动态库是否是libhwloc.so.15
