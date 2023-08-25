## 使用 OpenSSL 构建 SGFuzz 的详细教程

### 1. 构建并安装 SGFuzz
 ```shell
git clone https://github.com/bajinsheng/SGFuzz
cd SGFuzz
./build.sh 
cp libsfuzzer.a /usr/lib/libsFuzzer.a
 ```

### 2. 构建并安装 Honggfuzz
由于我们使用的是 hongfuzz 的网络驱动程序，我们还需要从 hongfuz 构建和安装网络驱动程序组件。
 ```shell
git clone https://github.com/google/honggfuzz.git && \
    cd honggfuzz && \
    git checkout 6f89ccc9c43c6c1d9f938c81a47b72cd5ada61ba && \
    CC=clang-10 CFLAGS="-fsanitize=fuzzer-no-link -fsanitize=address" make libhfcommon/libhfcommon.a && \
    CC=clang-10 CFLAGS="-fsanitize=fuzzer-no-link -fsanitize=address -DHFND_RECVTIME=1" make libhfnetdriver/libhfnetdriver.a && \
    mv libhfcommon/libhfcommon.a /usr/lib/libhfcommon.a && \
    mv libhfnetdriver/libhfnetdriver.a /usr/lib/libhfnetdriver.a
 ```

### 3. 指定 OpenSSL 使用的 tcp 端口
我们需要告诉网络驱动程序 OpenSSL 使用的 tcp 端口
```shell
export HFND_TCP_PORT=4433
```

### 4. 下载 OpenSSL
```shell
git clone https://github.com/openssl/openssl.git openssl-sfuzzer && \
    cd openssl-sfuzzer && \
    git checkout c74188e
```
### 5. 插桩状态变量
```
cd ../ && \
python3 {PATH}/SGFuzz/sanitizer/State_machine_instrument.py {PATH}/openssl-sfuzzer/ -b blocked_variables.txt
```
`-b` 选项是可选的。它用于阻止一些变量，这些变量可能不是枚举变量，并可能导致编译失败或其他问题。需要手动指定该列表。

### 6. 调整网络驱动程序的主要函数
```
cd openssl-sfuzzer && \
sed -i "s/ main/ HonggfuzzNetDriver_main/g" apps/openssl.c
```
这一步骤对于采用网络驱动程序是必要的。

### 7. 编译 OpenSSL
```
CC=clang-10 CXX=clang++-10 CFLAGS="-fsanitize=fuzzer-no-link -fsanitize=address" ./config no-shared && \
    make build_generated && \
    make apps/openssl
```

### 8. 链接 SGFuzz
因为我们修改了主函数，所以编译将在最后的链接阶段失败。我们需要重新运行此步骤，以使用三个新参数链接 SGFuzz："-lsFuzzer-lhfnetdriver-lhfcommon"
```
clang++-10 -fsanitize=fuzzer-no-link -fsanitize=address -lsFuzzer -lhfnetdriver -lhfcommon \
        -pthread -m64 -Wa,--noexecstack -Qunused-arguments -Wall -O3 -L.   \
        -o apps/openssl \
        apps/lib/openssl-bin-cmp_mock_srv.o \
        apps/openssl-bin-asn1parse.o apps/openssl-bin-ca.o \
        apps/openssl-bin-ciphers.o apps/openssl-bin-cmp.o \
        apps/openssl-bin-cms.o apps/openssl-bin-crl.o \
        apps/openssl-bin-crl2pkcs7.o apps/openssl-bin-dgst.o \
        apps/openssl-bin-dhparam.o apps/openssl-bin-dsa.o \
        apps/openssl-bin-dsaparam.o apps/openssl-bin-ec.o \
        apps/openssl-bin-ecparam.o apps/openssl-bin-enc.o \
        apps/openssl-bin-engine.o apps/openssl-bin-errstr.o \
        apps/openssl-bin-fipsinstall.o apps/openssl-bin-gendsa.o \
        apps/openssl-bin-genpkey.o apps/openssl-bin-genrsa.o \
        apps/openssl-bin-info.o apps/openssl-bin-kdf.o \
        apps/openssl-bin-list.o apps/openssl-bin-mac.o \
        apps/openssl-bin-nseq.o apps/openssl-bin-ocsp.o \
        apps/openssl-bin-openssl.o apps/openssl-bin-passwd.o \
        apps/openssl-bin-pkcs12.o apps/openssl-bin-pkcs7.o \
        apps/openssl-bin-pkcs8.o apps/openssl-bin-pkey.o \
        apps/openssl-bin-pkeyparam.o apps/openssl-bin-pkeyutl.o \
        apps/openssl-bin-prime.o apps/openssl-bin-progs.o \
        apps/openssl-bin-rand.o apps/openssl-bin-rehash.o \
        apps/openssl-bin-req.o apps/openssl-bin-rsa.o \
        apps/openssl-bin-rsautl.o apps/openssl-bin-s_client.o \
        apps/openssl-bin-s_server.o apps/openssl-bin-s_time.o \
        apps/openssl-bin-sess_id.o apps/openssl-bin-smime.o \
        apps/openssl-bin-speed.o apps/openssl-bin-spkac.o \
        apps/openssl-bin-srp.o apps/openssl-bin-storeutl.o \
        apps/openssl-bin-ts.o apps/openssl-bin-verify.o \
        apps/openssl-bin-version.o apps/openssl-bin-x509.o \
        apps/libapps.a -lssl -lcrypto -ldl -pthread
```
### 9. 运行目标 
```
cd experiment/openssl-sfuzzer && \
./apps/openssl -close_fd_mask=3 ../in-tls -- s_server -key ../key.pem -cert ../cert.pem -4 -no_anti_replay 
```


### 10.Dockerfile
我们提供了一个 dockerfile 来运行上述所有命令。请注意，网络驱动程序需要 docker 中的权限。
```
 docker build -t openssl .
 docker run -it --privileged openssl /bin/bash
```
