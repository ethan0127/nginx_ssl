应用场景: 公司内部一些业务对安全性要求比较高，这些业务系统只允许特定的员工访问，而且要求浏览器安装证书登录， 对公司新入职人员开通证书，
         离职人员注销证书。

一、自建CA证书需要openssl的支持。如果linux下没有安装openssl 请先安装 openssl。

      修改并确认/etc/pki/tls/openssl.cnf文件的配置
      vim  /etc/pki/tls/openssl.cnf
      dir=/etc/pki/CA             # CA的工作目录
      database=$dir/index.txt      # 签署证书的数据记录文件，下面步骤会生成index这个文件
      new_certs_dir=$dir/newcerts   # 存放新签署证书的目录
      serial=$dir/serial           # 新证书签署号记录文件下，下面步骤会生成serial这个文件
      certificate=$dir/ca.crt        # CA的证书路径，下面步骤会对应生成ca.crt这个证书
      private_key=$dir/private/cakey.pem    #CA的私钥路径，下面步骤会生成这个密钥
      crlnumber       = $dir/crlnumber         # 吊销证书用，下面步骤会生成这个文件
      crl             = $dir/ca.crl                # 当前证书列表,与下面步骤生成的吊销证书文件名一致




以下步骤（二、三、四）注意事项:
1、制作证书时会提示输入密码，设置密码可选，服务器证书和客户端证书密码可以不相同。
2、服务器证书和客户端证书制作时提示输入省份、城市、域名信息等，需保持一致，common Name这个需要不同。


二、使用openssl制作的CA子签名证书
    # 进入CA的工作目录
    cd /etc/pki/CA
    # 修改默认创建文件的权限
    umask077
    # 制作CA私钥
    openssl genrsa -out private/cakey.pem 2048
    #制作自签名证书
    openssl req -new -x509 -key private/cakey.pem -out ca.crt 
    # 生成数据记录文件，生成签署号记录文件，给文件一个初始号 
    touch index.txt
    touch serial
    touch crlnumber   ###注销证书用
    echo 01 > serial
    echo 01 > crlnumber
    openssl ca -gencrl -crldays3650 -out ca.crl # 产生crl文件，供吊销证书使用
    # 自建CA证书完成

三、 生产服务端证书
    cd /etc/pki/CA
    mkdir server
    cd server
    # umask077 权限设置
    # 制作服务器端私钥
    openssl genrsa -out server.key 1024
    # 制作服务器端证书申请指定使用sha512算法签名（默认使用sha1算法）
    openssl req -new -key server.key -sha512 -out server.csr 
    # 签署证书，3650是指证书有效期为10年
    openssl ca -in server.csr -out server.crt -days 3650
    # 完成服务器证书制作


四、生成客户端证书 (给客户端浏览器使用)
    # 创建并进入目录
    cd /etc/pki/CA
    mkdir client
    cd client
    # umask077 #设置权限

    # 制作客户端证书申请
    openssl req -new -key client.key -out client.csr
    # 签署证书 
    openssl ca -in client.csr -out client.crt -days 3650


五，nginx 配置
    server{
            listen 443;
            server_name localhost;
            ssi on;
            ssi_silent_errors on;
            ssi_types text/shtml;
            ssl on;
            ssl_certificate /etc/pki/CA/server/server.crt;
            ssl_certificate_key /etc/pki/CA/server/server.key;
            ssl_client_certificate /etc/pki/CA/ca.crt;
            ssl_verify_client on;
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
            ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
            ssl_prefer_server_ciphers On;
             ssl_crl /etc/pki/CA/server/ca.crl;  # 检查吊销的证书
            location / {
                proxy_pass   http://xxxx; 
                  }
    }
    注意: nginx配置里面各证书文件的路径请设置为自己生成的证书文件的路径。

    修改完nginx配置后，重启或者 重载配置文件 nginx -s reload 使新的配置生效

六、转换客户端证书格式，提供给浏览器使用
    # 进入客户端证书目录
    cd /etc/pki/CA/client
    # 转换格式，生成.p12文件
    openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12


七、撤销用户证书
    可以在index.txt看到 全部授权的证书。
    [root@master CA]# cat index.txt
    V 280117065511Z 01 unknown /C=CN/ST=ZheJiang/O=Default Company Ltd/CN=www.ethan.com
    R 280117070526Z 180119081337Z 02 unknown /C=CN/ST=ZheJiang/O=Default Company Ltd/CN=www.ethan.com
    V 280117072952Z 03 unknown /C=CN/ST=ZheJiang/O=zhanqi/OU=zq/CN=ethan
    R 280117073038Z 180119081329Z 04 unknown /C=CN/ST=ZheJiang/O=zhanqi/OU=zq/CN=yan


下面来注销第一行这个证书:
    cd newcerts
    [root@master newcerts]# ls
    01.pem 02.pem 03.pem 04.pem
    [root@master newcerts]# openssl ca -revoke 01.pem # index.txt 第一行证书对应文件为01.pem
    Using configuration from /etc/pki/tls/openssl.cnf
    Revoking Certificate 01.
    Data Base Updated
    再回头查看index.txt文件 发现第一行 之前的V变成了R (这应该是个标记位)。 说明完成该证书的注销
    [root@master newcerts]# cat ../index.txt
    R 280117065511Z 180119081733Z 01 unknown /C=CN/ST=ZheJiang/O=Default Company Ltd/CN=www.ethan.com
    R 280117070526Z 180119081337Z 02 unknown /C=CN/ST=ZheJiang/O=Default Company Ltd/CN=www.ethan.com
    V 280117072952Z 03 unknown /C=CN/ST=ZheJiang/O=zhanqi/OU=zq/CN=ethan
    R 280117073038Z 180119081329Z 04 unknown /C=CN/ST=ZheJiang/O=zhanqi/OU=zq/CN=yan

    # 进入目录 /etc/pki/CA
    cd /etc/pki/CA
    # 更新ca.crl文件
    openssl ca -gencrl -crldays 3650 -out ca.crl 

    # 重启nginx，使吊销证书操作生效。之后被吊销的证书将不能访问站点。








