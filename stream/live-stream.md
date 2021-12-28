# 直播流在web端、APP端、H5端观看的解决方案：摄像头观看、桌面共享

### 需求

摄像头在网页上回显

### 方案

1、在nginx上添加 `nginx-http-flv-module`模块，此模块可以接收 **rtmp 协议的流 并转换为 flv视频流**，再**通过  flv.js 在网页端播放**，APP中也可以播放

2、拉取摄像头的h264的视频流（一般rtsp协议）， 通过**ffmpeg将rtsp协议转换为rtmp协议并推送给nginx**

以上方案经测试延迟在3-10秒

### 安装&使用

#### nginx（centos）

```shell
cd /opt/nginx
#下载 nginx-http-flv-module 模块
yum install git
git clone https://github.com/winshining/nginx-http-flv-module.git
#安装依赖
yum install gcc make pcre-devel openssl-devel
#下载nginx 并解压
wget http://nginx.org/download/nginx-1.15.0.tar.gz
tar xf nginx-1.15.0.tar.gz
#进入nginx目录
cd nginx-1.15.0
#编译并安装
./configure --with-http_ssl_module --add-module=../nginx-http-flv-module && make && make install
```

#### nginx配置(服务器ip：192.168.1.15)

```nginx
rtmp_auto_push on;
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp;
 
rtmp {
    out_queue           4096;
    out_cork            8;
    max_streams         128;
    timeout             15s;
    drop_idle_publisher 15s;
	
    log_interval 5s; #log模块在access.log中记录日志的间隔时间，对调试非常有用
    log_size     1m; #log模块用来记录日志的缓冲区大小
	
    server {
        listen 1935;                               #监听端口
        chunk_size 4000;                           #包大小，默认4096，值越大，CPU越低，不能小于128

        application forword {                      #rtmp推流请求路径
			live on;                               #开启直播
			#gop_cache on; #打开GOP缓存，减少首屏等待时间
        }
    }
}

http {

    ...

    server {
        listen       80;
        
        ...
        
        location /live {
            flv_live on; #打开 HTTP 播放 FLV 直播流功能
            chunked_transfer_encoding on; #支持 'Transfer-Encoding: chunked' 方式回复
 
            add_header 'Access-Control-Allow-Origin' '*'; #支持跨域请求
            add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的 HTTP 头
            add_header 'Cache-Control' 'no-cache';
        }
        
        ...
		
    }
}
```

#### ffmpeg命令

```shell
ffmpeg -i "rtsp://admin:zhihe123@192.168.1.203:554/h264/ch1/main/av_stream" -vcodec libx264 -f flv -tune zerolatency -preset ultrafast "rtmp://192.168.1.15:1935/forword/xx"
```

#### flv播放地址

`http://192.168.1.15/live?port=1935&app=forword&stream=xx`

#### flv.js使用（PC端可以，IOS的H5不支持，安卓H5未测试）

```javascript
if (flvjs.isSupported()) {
    startVideo()
}

function startVideo(){
    var videoElement = document.getElementById('videoElement');
    var flvPlayer = flvjs.createPlayer({
        type: 'flv',
        isLive: true,
        url: 'http://192.168.1.15/live?port=1935&app=forword&stream=xx'
    });
    flvPlayer.attachMediaElement(videoElement);
    flvPlayer.load();
    flvPlayer.play();
}
```

[demo](./flv-demo.html)

#### uni-app使用(安卓APP端可以使用，IOS的H5不支持，安卓H5未测试)

```vue
<template>
  <div class="viedo">
    <video
     id="video1"
     controls=false
     preload="auto"
     style="width:100%"
     :src="hlsDownAddress"
     show-play-btn=false
     enable-progress-gesture=false
     autoplay=true
     direction=90
     play-strategy=3
    >
    <button type="default" @click="change2">摄像头</button>
  </div>
</template>
<script>
	export default {
		data() {
			return{
				hlsDownAddress:"http://192.168.1.15/live?port=1935&app=hls&stream=tt",
			}
		},
		methods:{
			change2:function(){
				this.hlsDownAddress = "http://192.168.1.15/live?port=1935&app=forword&stream=xx";
				this.videoContext1.play()
			}
		}
		onReady: function(res) {
			this.videoContext1 = uni.createVideoContext('video1')
		},
</script>
```

#### H5上播放的解决方案

通过nginx存储m3u8来实现播放，**缺点是延迟高**，实测在10-30秒

##### nginx配置

```nginx
rtmp {
    server {
        listen 1935;                               #监听端口
        chunk_size 4000;                           #包大小，默认4096，值越大，CPU越低，不能小于128

        application hls {                          #rtmp推流请求路径
            live on;                               #开启直播
            hls on;                                #开启hls
            hls_path /usr/share/nginx/html/hls;    #rtmp推流文件存放路径，要可读可写的权限
            hls_fragment 1s;                       #每个TS文件包含5秒的视频内容
            hls_playlist_length 5s;
        }
    }
}
http {
    
    ...

    server {
        listen       80;
		
        location / {
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Headers X-Requested-With;
            add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
            root  /usr/share/nginx/html;
        }
    }
}
```

**m3u8播放地址：`http://192.168.1.15/xx.m3u8`**

## 后期优化

- nginx播放m3u8文档不够完善

## 最后

以上方案也可以用于桌面共享，通过OBS录屏并推流给nginx（这部分可以二次开发OBS实现服务化，开机启动桌面共享，比如学校的PC机远程监看等需求），其他人可通过WEB端、APP端、H5端进行观看

