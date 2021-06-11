

## 确定源码目录 及安装目录

```bash
mkdir -p ~/Desktop/CProgram/grpc_rdma_0524
mkdir -p ~/Desktop/CProgram/grpc_rdma_0524/grpc-rdma-install-dir
```

## 指定编译使用到的环境变量

```bash
export MY_INSTALL_DIR=$HOME/Desktop/CProgram/grpc_rdma_0524/grpc-rdma-install-dir
```

## 获取源码

```bash
git clone -b main https://github.com/raygift/grpc-rdma.git grpc
```

- -b 指定clone 某个特定分支
- 末尾指定clone 的目标文件夹，可与仓库中项目文件名不同

## 获取submodule

```bash
cd grcp
git submodule udpate --init
```

- 若由于网络问题submodule 无法获取，开启lantern 后，设置proxy
  export https_proxy=http://127.0.0.1:45051

## 编译必要的submodule

### （必要）编译第三方依赖 abseil

*abseil 是谷歌的 C++ 库，作为 C++ 标准库的补充*

```bash
$ mkdir -p third_party/abseil-cpp/cmake/build
$ pushd third_party/abseil-cpp/cmake/build
$ cmake -DCMAKE_INSTALL_PREFIX=$MY_INSTALL_DIR \
      -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE \
      ../..
$ make
$ make install
$ popd
```



## (grpc-rdma 需要)添加rdma链接

CMakeList.txt 2022行添加如下库地址

```cmake
  /usr/lib/x86_64-linux-gnu/librdmacm.so
  /usr/lib/x86_64-linux-gnu/libibverbs.so
```



## 编译gRPC

```bash
cmake -DgRPC_INSTALL=ON \
      -DgRPC_BUILD_TESTS=OFF \
      -DCMAKE_INSTALL_PREFIX=$MY_INSTALL_DIR \
      ../..
make
make install
make clean
```



## 编译examples

```bash
cd examples/cpp/helloworld
mkdir -p cmake/build
pushd cmake/build/

export MY_INSTALL_DIR=$HOME/Desktop/CProgram/grpc_rdma_0524/grpc-rdma-install-dir
cmake -DCMAKE_PREFIX_PATH=$MY_INSTALL_DIR ../..

make
popd
```

### 配置Debug 环境变量

在命令行中定义如下环境变量，用以打开grpc debug 日志

```bash
export GRPC_VERBOSITY=DEBUG
export GRPC_TRACE=api,tcp,polling 
## 跟踪api的调用
```

