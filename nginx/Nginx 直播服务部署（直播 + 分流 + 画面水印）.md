# 5 分钟完成 Nginx 直播服务部署（直播 + 分流 + 画面水印）

### 前言

> 最近帮朋友的公司部署了一套分流+水印的直播系统
>
> 顺手打包成docker镜像，方便大家需要用到的时候开箱即用，不需要百度一些零碎的文章 也可做简单的直播服务，只需调整配置文件便可达到你的需求.
>
> 需求：将直播流分流到两个云厂商的直播云，一个有水印，一个无水印。使用hls播放
>
> 朋友需求的拓扑示意图：
>
> 
>
> ![ar414-nginx-rtmp](https://user-gold-cdn.xitu.io/2020/3/13/170d4824f9c72d64?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
>
> 
>
> 当前拓扑示意图（阿某云和腾讯云不方便放出推流和拉流地址，有兴趣的同学可以去申请玩一下）
>
> 
>
> ![ar414-nginx-service](https://user-gold-cdn.xitu.io/2020/3/13/170d4824fa1eb3ec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# [docker-nginx-rtmp-ffmpeg](https://hub.docker.com/repository/docker/ar414/nginx-rtmp-ffmpeg)

基于[docker-nginx-rtmp](https://github.com/alfg/docker-nginx-rtmp)进行配置部署，这篇文章的意义是实现直播分流及直播画面水印.

- Nginx 1.16.1（从源代码编译的稳定版本）

- nginx-rtmp-module 1.2.1（从源代码编译）

- ffmpeg 4.2.1（从源代码编译）

- 已配置好的

  nginx.conf

  - 只支持1920*1080（如需支持其他分辨率可参考[nginx.conf](https://github.com/alfg/docker-nginx-rtmp/blob/master/nginx.conf)）
  - 实现两路分流
    - 本机
    - 直播云（例：阿某云、腾讯云、ucloud）
  - 实现直播水印效果
    - 水印图片存放位置（容器内）：/opt/images/logo.png

## 部署运行

### 服务器

- 安装docker(Centos7,其他系统请发挥你的搜索功能)

```
$ yum -y install docker #安装docker
$ systemctl enable docker #配置开机启动
$ systemctl start docker #启动docker服务
复制代码
```

- 拉取docker镜像并运行

```
#如果速度慢可使用阿某云：docker pull registry.cn-shenzhen.aliyuncs.com/ar414/nginx-rtmp-ffmpeg:v1
$ docker pull ar414/nginx-rtmp-ffmpeg
$ docker run -it -d -p 1935:1935 -p 8080:80 --rm ar414/nginx-rtmp-ffmpeg
复制代码
```

- 推流地址（Stream live content to）：

```
rtmp://<server ip>:1935/stream/$STREAM_NAME
复制代码
```

- SSL证书

将证书复制到容器内，并在容器内修改nginx.conf配置文件，然后重新commit（操作容器内的文件都需要重新commit才会生效）

```
#/etc/nginx/nginx.conf
listen 443 ssl;
ssl_certificate     /opt/certs/example.com.crt;
ssl_certificate_key /opt/certs/example.com.key;
复制代码
```

### OBS配置

- Stream Type: `Custom Streaming Server`

- URL: `rtmp://:1935/stream`

- Stream Key：ar414

  ![obs-config](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="997" height="787"></svg>)

### 观看测试

> HLS播放测试工具：[player.alicdn.com/aliplayer/s…](http://player.alicdn.com/aliplayer/setting/setting.html) （如果配置了证书则使用https）

- HLS播放地址

  - 有水印：http://<server ip>:8080/hls/ar414_wm.m3u8

    ![ar414-hls-wm](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="892" height="531"></svg>)

  - 无水印：http://<server ip>:8080/hls/ar414.m3u8

    ![ar414-hls](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="899" height="534"></svg>)

> RTMP测试工具：[PotPlayer](https://daumpotplayer.com/download/)

- RTMP播放地址

  - 无水印：rtmp://<server ip>:1935/stream/ar414

    ![ar414-rtmp](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1280" height="720"></svg>)

  - 有水印：需要分流到其他服务器上

## :page_facing_up:配置文件简解（分流、水印及水印位置）

> [完整配置文件](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf/blob/master/nginx.conf)

- RTMP配置

```
rtmp {
    server {
        listen 1935; #端口
        chunk_size 4000;
        #RTMP 直播流配置
        application stream {
            live on;
            #添加水印及分流，这次方便测试直接分流到当前服务器hls
            #实际生产一般都分流到直播云（腾讯云、阿某云、ucloud）
            #只需把需要分流的地址替换即可
            #有水印：rtmp://localhost:1935/hls/$name_wm
            #无水印：rtmp://localhost:1935/hls/$name
            exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
              -filter_complex "overlay=10:10,split=1[ar414]"
              -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost:1935/hls/$name_wm
              -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://localhost:1935/hls/$name;
        }

        application hls {
            live on;
            hls on;
            hls_fragment 5;
            hls_path /opt/data/hls;
        }
    }
}
复制代码
```

- 如果需要推多个直播云则复制多个 exec ffmpeg即可 如下：

```
application stream {
    live on;
    #分流至本机hls           
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost:1935/hls/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://localhost:1935/hls/$name;
    
    #分流至腾讯云
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.tencent.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.tencent.com/stream/$name;

    #分流至阿某云
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.aliyun.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.aliyun.com/stream/$name;
}
复制代码
```

- 水印位置

  - 水印位置

    | 水印图片位置 | overlay值                                 |
    | ------------ | ----------------------------------------- |
    | 左上角       | 10:10                                     |
    | 右上角       | main_w-overlay_w-10:10                    |
    | 左下角       | 10:main_h-overlay_h-10                    |
    | 右下角       | main_w-overlay_w-10 : main_h-overlay_h-10 |

  - overlay参数

    | 参数      | 说明                                 |
    | --------- | ------------------------------------ |
    | main_w    | 视频单帧图像宽度（当前配置文件1920） |
    | main_h    | 视频单帧图像高度（当前配置文件1080） |
    | overlay_w | 水印图片的宽度                       |
    | overlay_h | 水印图片的高度                       |

## 结语



作者：码怪鲁迅
链接：https://juejin.im/post/5e6ba67651882549652d67ec
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。