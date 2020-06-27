# KVM 安装

## install vnc

```shell
yum install tigervnc -y
```



--------------------------------------

## compile kvm

```shell
yum install ncurses-devel -y
yum install openssl* -y
yum install elfutils-libelf* -y
make all    
make modules_install
make install
```



-------------------------------------

## complie & install qemu

```shell
git clone https://git.qemu.org/git/qemu.git
cd qemu
git submodule init
git submodule update --recursive

yum install python3* -y
yum install gtk2-devel -y

./configure
make
make install
```



--------------------------------------

## install virt manager

```shell
yum install virt-manager -y
```



-------------------------------------