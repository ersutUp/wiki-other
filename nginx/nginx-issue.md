# nginx 问题&坑

----------

## <div id="win-nginx"></div>windows中部署nginx后发现请求达到870左右报500

### 报错信息
`maximum number of descriptors supported by select() is 1024 while waiting`

### 处理
1. 将 work_connections 调大后依然无效。
2. nginx.org所下载的nginx在windows中只支持1024个tcp链接(实测更少)，即使将work_connections 调大也没有
3. 在http://nginx-win.ecsds.eu/下载 windows版nginx，运行nginx_base.exe，问题解决

## <div id="error-level"></div>nginx的错误日志等级

error_log 级别分为 debug, info, notice, warn, error, crit  默认为crit。

格式：
error_log  xxx/error.log [level];

