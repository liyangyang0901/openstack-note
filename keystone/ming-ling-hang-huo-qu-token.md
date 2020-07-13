# 命令行获取token

#### 问题

调试restAPI时需要获取token

#### 解决方案

通过CURL在命令行进行http请求获取token

```text
curl -XPOST -i \
  --url http://ip:port/v3/auth/tokens \
  --header 'content-type: application/json' \
  --data '{"auth": {"identity": {"methods": ["password"],"password": {"user": {"name": "admin","domain": {"name": "Default"},"password": "xxx"}}},"scope": {"project": {"domain": {"id": "default"},"name": "admin"}}}}'
```

在响应的header中可以找到token

