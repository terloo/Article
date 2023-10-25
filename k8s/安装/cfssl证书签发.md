# 证书签发

## 配置关于证书签发的工具
1. 下载安装`cfssl cfssljson cfssl-certinfo`
    ```shell
    wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
    chmod +x cfssl*
    cp cfssl_linux-amd64 /usr/local/bin/cfssl
    cp cfssljson_linux-amd64 /usr/local/bin/cfssljson
    cp cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
    ```
2. 配置证书生成策略，规定CA可以颁发那种类型的证书
    `vim ca-config.json`
    ```json
    {
        "signing": {
            "default": {
                "expiry": "8760h"
            },
            "profiles": {
                "kubernetes": {
                    "usages": ["sign", "key encipherment", "server auth", "client auth"],
                    "expiry": "8760h"
                }
            }
        }
    }
    // 8760h为证书有效期（一年），可自行修改
    // 这里是设置一个名为kubernetes的ca，并规定其颁布的证书有效时间和作用范围，以供k8s使用
    ```

## 向CA请求证书
1. 生成CA的证书和私钥
   1. 编写CA证书生成请求体`vim ca-csr.json`
      ```json
      {
          "CN": "Kubernetes",  // Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法。
          "hosts": [],  // 准许使用的域名，为空时为全部允许
          "key": {
              "algo": "rsa",
              "size": 2048
          },
          "names": [
              {
                  "C": "CN",  // 国家 
                  "ST": "SiChuan",  // 省份（州）
                  "L": "ChengDu", // 城市
                  "O": "CA",  // 组织 
                  "OU": "System"  // 组织单元
              }
          ]
      }
      ```
   2. 生成CA证书和私钥
      ```shell
      cfssl gencert -initca ca-csr.json | cfssljson -bare ca
      ```
      得到`ca.pem`(ca证书)、`ca.csr`(签名请求)、`ca-key.pem`(ca私钥)

2. 向CA申请一个管理员admin使用的证书
   1. 编写证书请求体`vim admin-csr.json`
      ```json
      {
          "CN": "admin",  // kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)
          "hosts": [],
          "key": {
              "algo": "rsa",
              "size": 2048
          },
          "names": [
              {
                  "C": "CN",  // 国家 
                  "ST": "SiChuan",  // 省份（州）
                  "L": "ChengDu", // 城市
                  "O": "system:master",  // 组织 kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
                  "OU": "System"  // 组织单元
              }
          ]
      }
      ```
   2. 产生证书
      ```shell
      cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
            -config=ca-config.json -profile=kubernetes \
            admin-csr.json | cfssljson -bare admin
      ```

3. 向CA申请一个kubelet的证书
   1. 编写证书请求体`vim zhangzeyu-kubelet-csr.json`，**注意修改用户名和组**
      ```json
      {
          "CN": "system:node:zhangzeyu",  // 主机名
          "hosts": [],
          "key": {
              "algo": "rsa",
              "size": 2048
          },
          "names": [
              {
                  "C": "CN",
                  "ST": "SiChuan",
                  "L": "ChengDu",
                  "O": "system:nodes",
                  "OU": "System"
              }
          ]
      }
      ```
   2. 产生证书，**这里还需要配置hostname来使证书和节点一对一**
      ```shell
      cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
            -config=ca-config.json -profile=kubernetes \
            -hostname=zhangzeyu,127.0.0.1 \
            zhangzeyu-kubelet-csr.json | cfssljson -bare zhangzeyu-kubelet
      ```

4. 向CA申请一个controller-manager证书
   1. 编写证书请求体`vim kube-controller-manager-csr.json`
      ```json
      {
          "CN": "system:kube-controller-manager",
          "hosts": [],
          "key": {
              "algo": "rsa",
              "size": 2048
          },
          "names": [
              {
                  "C": "CN",
                  "ST": "SiChuan",
                  "L": "ChengDu",
                  "O": "system:kube-controller-manager",
                  "OU": "System"
              }
          ]
      }
      ```
   2. 产生证书
      ```shell
      cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
            -config=ca-config.json -profile=kubernetes \
            kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
      ```
