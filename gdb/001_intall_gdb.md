## 源码安装

### 1. 下载 gdb 版本
```
 wget http://ftp.gnu.org/gnu/gdb/gdb-11.2.tar.gz
```
### 2.  解压
```
tar -xvf gdb-11.2.tar.gz
```
### 3. 编译并安装
```
mkdir build && cd build
../configure --prefix={你想要安装的路径}/install
make -j16
make install
```
### 4. 添加 bin 路径到 PATH
```
export PATH={你想要安装的路径}/install/bin:$PATH
```

### 5. Q & A

Q: makeinfo is missing on your system

A: sudo apt install texinfo


## 仓库安装

### 1. apt 源安装
```
sudo apt install gdb
```
### 2. yum 源安装
```
sudo yum install gdb
```