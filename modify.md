
1.try remove server TLS instead by Nginx no need we use custom inbound and out bound
服务器端：
Trojan 配置文件例子：
```json
{
  "run_type": "custom",
  "inbound": {
    "node": [
      {
        "protocol": "websocket",
        "tag": "websocket",
        "config": {
          "remote-addr": "127.0.0.1",
          "remote-port": 80,
          "websocket": {
            "enabled": true,
            "path": "/class/",
            "host": "vvdd.eu.org"
          }
        }
      },
      {
        "protocol": "transport",
        "tag": "transport",
        "config": {
          "local-addr": "0.0.0.0",
          "local-port": 14433,
          "remote-addr": "127.0.0.1",
          "remote-port": 80
        }
      },
      {
        "protocol": "mux",
        "tag": "mux",
        "config": {
          "enabled": true,
          "concurrency": 8,
          "idle_timeout": 60
        }
      },
      {
        "protocol": "trojan",
        "tag": "trojan1",
        "config": {
          "remote-addr": "127.0.0.1",
          "remote-port": 80,
          "password": [
            "123456"
          ]
        }
      }
    ],
    "path": [
      [
        "transport",
        "websocket",
        "mux",
        "trojan1"
      ]
    ]
  },
  "outbound": {
    "node": [
      {
        "protocol": "freedom",
        "tag": "freedom"
      }
    ],
    "path": [
      [
        "freedom"
      ]
    ]
  }
}

```
Nginx 配置例子：
```
map $http_upgrade $connection_upgrade {
default upgrade;
'' close;
}

server {
listen 80;
listen [::]:80;

        server_name vvdd.eu.org;

        listen 443 ssl;
        listen [::]:443 ssl;

        ssl_certificate /etc/letsencrypt/live/vvdd.eu.org/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/vvdd.eu.org/privkey.pem;
        ssl_protocols TLSv1.3;

        root /var/www/vvdd.eu.org/;
        index index.html;

        location / {
                try_files $uri $uri/ =404;
        }
        location /class/ {
                proxy_pass  http://127.0.0.1:14433; # 转发规则，注意最后的斜杠
                proxy_set_header Host $proxy_host; # 修改转发请求头，让8080端口的应用可以受到真实的请求
                proxy_set_header X-Real-IP $remote_addr;
                proxy_read_timeout 60s;
                proxy_send_timeout 60s;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
        }
}
```

客户端：
Trojan 配置例子：

```json
{
  "run_type": "custom",
  "inbound": {
    "node": [
      {
        "protocol": "adapter",
        "tag": "adapter",
        "config": {
          "local-addr": "127.0.0.1",
          "local-port": 1082
        }
      },
      {
        "protocol": "socks",
        "tag": "socks",
        "config": {
          "local-addr": "127.0.0.1",
          "local-port": 1082
        }
      }
    ],
    "path": [
      [
        "adapter",
        "socks"
      ]
    ]
  },
  "outbound": {
    "node": [
      {
        "protocol": "router",
        "tag": "router",
        "config": {
          "router": {
            "enabled": true,
            "bypass": [
              "geoip:cn",
              "geoip:private",
              "geosite:private",
              "geosite:geolocation-cn"
            ],
            "block": [
              "geosite:category-ads"
            ],
            "proxy": [
              "geosite:geolocation-!cn",
              "domain:www.tiktok.com",
              "domain:tiktok.com"
            ],
            "default_policy": "proxy",
            "geoip": "/Users/victor/Downloads/trojan-go-darwin-amd64/geoip.dat",
            "geosite": "/Users/victor/Downloads/trojan-go-darwin-amd64/geosite.dat"
          }
        }
      },
      {
        "protocol": "transport",
        "tag": "transport",
        "config": {
          "remote-addr": "vvdd.eu.org",
          "remote-port": 443
        }
      },
      {
        "protocol": "tls",
        "tag": "tls",
        "config": {
          "ssl": {
            "verify": true,
            "verify_hostname": true,
            "sni": "vvdd.eu.org"
          }
        }
      },
      {
        "protocol": "websocket",
        "tag": "websocket",
        "config": {
          "websocket": {
            "enabled": true,
            "path": "/class/",
            "host": "vvdd.eu.org"
          }
        }
      },
      {
        "protocol": "mux",
        "tag": "mux",
        "config": {
          "enabled": true,
          "concurrency": 8,
          "idle_timeout": 60
        }
      },
      {
        "protocol": "trojan",
        "tag": "trojan",
        "config": {
          "password": [
            "123456"
          ]
        }
      }
    ],
    "path": [
      [
        "router",
        "transport",
        "tls",
        "websocket",
        "mux",
        "trojan"
      ]
    ]
  }
}
```