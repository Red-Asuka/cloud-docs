# 配置 TLS/SSL

>该功能在免费试用和共享部署中不可用



## TLS/SSL 介绍
作为基于现代密码学公钥算法的安全协议，TLS/SSL 能在计算机通讯网络上保证传输安全。
TLS/SSL 带来的安全优势
- 强认证。 用 TLS 建立连接的时候，通讯双方可以互相检查对方的身份。在实践中，很常见的一种身份检查方式是检查对方持有的 X.509 数字证书。这样的数字证书通常是由一个受信机构颁发的，不可伪造。

- 保证机密性。TLS 通讯的每次会话都会由会话密钥加密，会话密钥由通讯双方协商产生。任何第三方都无法知晓通讯内容。即使一次会话的密钥泄露，并不影响其他会话的安全性。

- 完整性。 加密通讯中的数据很难被篡改而不被发现。

EMQ X Cloud 支持两种方式的认证：单向认证和双向认证<br>
单向认证需要：
- 服务器证书
- 证书链
- 私有秘钥

双向认证需要：
- 服务器证书
- 证书链
- 私有秘钥
- 客户端 CA 证书

## 证书限制
- 证书必须指定加密算法和密钥大小。EMQ X Cloud 支持下列算法：
  - 1024 位 RSA (RSA_1024)
  - 2048 位 RSA (RSA_2048)
- 证书必须是 SSL/TLS X.509 版本 3 证书。它必须包含公有密钥、网站的完全限定域名 (FQDN) 或 IP 地址以及有关发布者的信息。证书可以由您的私有密钥或发证 CA 的私有密钥进行自签名。如果证书由 CA 签名，则在导入证书时必须包含证书链。
- 证书必须是有效的，有效期开始之前和结束的 `180` 天之内，无法导入证书。
- 证书、私有密钥和证书链必须采用 PEM 编码。
- 私有秘钥必须是无密码的。
- 证书的加密算法必须与签名 CA 的加密算法匹配。例如，如果签名 CA 的密钥类型为 RSA，则该证书的密钥类型也必须为 RSA。

### 格式说明
#### 证书格式
```
-----BEGIN CERTIFICATE-----
Base64–encoded certificate
-----END CERTIFICATE-----
```

#### 证书链格式
> 不要把证书复制到证书链中

```
-----BEGIN CERTIFICATE-----
Base64–encoded certificate
-----END CERTIFICATE-----
```

#### 私有秘钥格式
```
-----BEGIN RSA PRIVATE KEY-----
Base64–encoded private key
-----END RSA PRIVATE KEY-----
```

### 自签名证书
请先确保您已经安装了 [OpenSSL](https://www.openssl.org/)
#### 创建服务端 CA 证书
```bash
openssl req \
    -new \
    -newkey rsa:2048 \
    -days 3650 \
    -nodes \
    -x509 \
    -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=CA" \
    -keyout cloud-ca.key \
    -out cloud-ca.crt
```

生成的 `cloud-ca.crt` 为服务端 CA 证书，用于签发服务端证书和连接时验证服务端证书。

#### 创建服务端证书

生成私钥
```bash
openssl genrsa -out cloud.key 2048
```
创建 `openssl.cnf` 文件，修改 `alt_names` 的 `CONNECT_ADDRESS` 为部署详情界面的连接地址。`req_distinguished_name` 根据需求进行修改。
```
[req]
default_bits  = 2048
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
countryName = CN
stateOrProvinceName = XX
localityName = XX
organizationName = XX
commonName = XX
[req_ext]
subjectAltName = @alt_names
[v3_req]
subjectAltName = @alt_names
[alt_names]
IP.1 = CONNECT_ADDRESS
```

生成证书请求文件 `cloud.csr`
```bash
openssl req -new -key cloud.key -config openssl.cnf -out cloud.csr 
```

用 CA 证书给服务端签名
```bash
openssl x509 -req -days 3650 -in cloud.csr -CA cloud-ca.crt -CAkey cloud-ca.key -CAcreateserial -out cloud.crt  -extensions v3_req -extfile openssl.cnf 
```

上面步骤主要生成了以下文件：
- cloud.crt：服务端证书
- cloud.key：私有秘钥

#### 创建客户端证书(双向认证)
```bash
# 创建客户端 CA 证书
openssl req \
    -new \
    -newkey rsa:2048 \
    -days 3650 \
    -nodes \
    -x509 \
    -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=Client CA" \
    -keyout client-ca.key \
    -out client-ca.crt

# 创建客户端 key
openssl genrsa -out client.key 2048
# 创建客户端请求文件
openssl req -new -key client.key -out client.csr -subj "/C=CN/ST=XX/L=XX/O=EMQ X Cloud/CN=Client"
# 用 CA 证书给客户端签名
openssl x509 -req -days 3650 -in client.csr -CA client-ca.crt -CAkey client-ca.key -CAcreateserial -out client.crt
```
以上步骤主要生成以下文件：
- client-ca.crt：客户端 CA 证书
- client.key：客户端私有秘钥
- client.crt：客户端证书

## 创建证书
1. 创建部署，并等待至 `running` 状态。
2. 登录 [EMQ X Cloud 控制台](<https://cloud.emqx.io/console>)。
3. 进入部署详情，点击 `+TLS/SSL 配置` 按钮，配置证书内容，您可以上传文件或者直接填写证书内容
   - 认证类型：
     - 单向认证：仅客户端验证服务端证书
     - 双向认证：客户端和服务端相互验证证书
   - 证书：服务端证书
   - 证书链：证书链，通常第三方机构签发证书时会提供
   - 私有秘钥：私有秘钥
   - CA 证书：选择双向认证时，需要提供客户端的 CA 证书
4. 填写完成后，点击“确定”。

![tls](../_assets/deployments/tls.png)

## 测试连接
测试之前，请确保创建了认证信息，参考 [认证和鉴权](./users_and_acl.md)。<br>
您可以使用 [MQTTX](<https://mqttx.app/>) 连接和测试。<br>
我们使用 MQTTX 进行测试：
- 新建连接，输入 Name，Client ID 随机生成即可
- 选择 Host，填入部署的连接地址和端口
  - 若选择 SSL 连接，选择 `mqtts://` 和 `8883` 端口
  - 若选择 Websocket with SSL，选择 `wss://` 和 `8084` 端口
- 输入创建的认证信息：用户名和密码
- SSL/TLS 选择 true
- 证书选择
  - 第三方机构认证的证书，不需要提供 CA 证书
  - 自签名证书，提供服务端 CA 证书，若双向认证，还需要提供客户端证书和私钥
- 打开严格模式
- 连接

![mqttx_tls](../_assets/deployments/mqttx_tls.png)

## 删除证书
删除证书会断开客户端到 `8883` 和 `8084` 的连接，请确保这不会影响到您的业务。
1. 登录 [EMQ X Cloud 控制台](<https://cloud.emqx.io/console>)。
2. 进入部署详情，点击 `TLS/SSL 配置` 部分的证书的删除按钮。
3. 在对话框点击“确定”，完成删除。

## 常见问题

1. 证书内容包含多个证书
  
   购买的证书包含中间证书，以文本形式打开证书，多个证书是按照用户证书 - 中间证书 - Root 证书的顺序。一般来说，证书包含用户证书和多个中间证书，您需要将用户证书和中间证书分离开，并将中间证书填入到证书链中。
   ```
   -----BEGIN CERTIFICATE-----

   用户证书
   
   -----END CERTIFICATE-----
   
   -----BEGIN CERTIFICATE-----
   
   中间证书
   
   -----END CERTIFICATE-----
   
   -----BEGIN CERTIFICATE-----
   
   根证书
   
   -----END CERTIFICATE-----
   ```
2. 缺少证书链
  
    证书链补全：https://myssl.com/chain_download.html