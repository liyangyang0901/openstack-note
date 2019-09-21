# nova-consoleauth:token无效.md

\#\#\# 问题

Fuel9部署的M版OpenStack，连接控制台时经常出现\`Failed to connect to server \(code: 1006\)\`。三台控制节点的情况下，要将页面刷新3次才能正常连接。

\#\#\# 原因

nova服务配置问题，Fuel9在nova服务中配置的memcache有误，导致memcached未生效，nova-consoleauth实际使用了本地缓存。nova-consoleauth生成的token只保存在了处理请求的单个节点上，等到浏览器端发起连接时，token验证失败。如果有3个控制节点，按照轮询策略，刚好三次才能轮询到持有token的节点上。

\#\#\# 解决方案

修改\`/etc/nova/nova.conf\`

增加以下配置

\`\`\`

\[cache\]

enabled = true

backend = dogpile.cache.memcached

\#注意没有d

memcache\_servers = controller1:11211,controller1:11211,controller1:11211

\`\`\`

\#\#\# 相关链接

bug描述

\[https://bugs.launchpad.net/fuel/+bug/1621930\]\(https://bugs.launchpad.net/fuel/+bug/1621930\)

