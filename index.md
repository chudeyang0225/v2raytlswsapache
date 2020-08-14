**V2ray+tls+ws (Centos7, Apache)**

# 申请证书
使用acme.sh申请域名证书，需要独占80端口，所以在下面步骤中会禁用apache监听80端口、或设置在申请/更新时自动临时关闭apache服务器。
## 安装
```
yum -y install netcat
curl https://get.acme.sh | sh
source ~/.bashrc
```
## 申请
```
~/.acme.sh/acme.sh --issue -d abc.mydomain.net --standalone -k ec-256
```
需要自动临时关闭apache服务器并在事后自动重启，需要在这行命令添加pre-hook和post-hook。这里添加的hook会自动生效到更新证书的cronjob里。如果这里没有添加，事后无法在cronjob中手动添加。
```
~/.acme.sh/acme.sh --issue -d abc.mydomain.net --standalone -k ec-256 --pre-hook "systemctl stop httpd" --post-hook "systemctl restart httpd"
```
## 安装证书到/etc/v2ray/文件夹
安装位置及证书名都可以自定义，在下文Apache设置中需要对应
安装后将来的更新证书不需要手动再安装一次
```
~/.acme.sh/acme.sh/ --installcert -d abc.mydomain.net --fullchainpath /etc/v2ray/v2ray.crt --keypath /etc/v2ray/v2ray.key --ecc
```

# 配置Apache(httpd)
## 安装
```
yum -y install httpd
yum -y instsall mod_ssl 
```

## 设置
1. 修改httpd.conf  `/etc/httpd/conf/httpd.conf`
注释掉`listen port 80`避免证书申请/更新时端口冲突
文末添加两行：
```
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule ssl_module modules/mod_ssl.so
```
2. 添加域名配置: 新建 `/etc/httpd/conf.d/abc.yourdomain.net.conf`
**此处3579为内部转发使用的任选端口号**  
**此处/ray 为apache转发v2ray流量使用的识别符，后续v2ray客户端中需要对应**  


```
<VirtualHost *:443>
    DocumentRoot /var/www/html/
    ServerName abc.yourdomain.net  //自定义域名
    SSLEngine On
    RewriteEngine On
        RewriteCond %{HTTP:Upgrade} =websocket [NC]
        RewriteRule /(.*)    ws://127.0.0.1:3579/$1 [P,L]
    SSLProxyEngine On
    Proxypass /ray http://localhost:3579
    ProxyPassReverse /ray http:localhost:3579
    SSLCertificateFile /etc/v2ray/v2ray.crt
    SSLCertificateKeyFile /etc/v2ray/v2ray.key
</VirtualHost>
```

# V2Ray设置
## 服务端
```
{
    "log" : {
      "access": "/var/log/v2ray/access.log",
      "error": "/var/log/v2ray/error.log",
      "loglevel": "warning"
  },
    "inbound": {
      "port": 3579, //同Apache中的内部转发端口
      "listen":"127.0.0.1", //只监听本地流量
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": “UUID”,
            "level": 1,
            "alterId": 64
          }
          ]
      },
      "streamSettings":{
        "network":"ws",
        "wsSettings":{
          "path":"/ray" //同apache中流量识别符
          }
        }
      },
  "inboundDetour":[  //此段可选，不走tls+ws的渠道
    {
      "protocol": "vmess",
      "port": 33634,
      "settings":{
         "clients": [
          {
            "id": “UUID”,
            "level": 1,
            "alterId": 64
          }
          ]
       },
      "streamSettings":{
        "network": "tcp"
      }
    },
    {
      "protocol": "shadowsocks",
      "port": 20037,
      "settings": {
         "method": "aes-256-cfb",
         "password": “PASSWORD”,
         "network": "tcp,udp",
         "level": 1
      }
    }
  ],
  "outbound": {
    "protocol": "freedom",
    "settings": {}
    }
}
```
