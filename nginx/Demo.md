# Demo

## 防盗链
```conf
server {
    listen 9999;
    server_name localhost;
    location / {
        root /opt/test/;
    }
    
    location ~* .*\.(gif|jpg|png)$ {
        root /opt/test/img/;
        valid_referers none block taobao.com;
        if ($invalid_referer) {
            # return 403
            rewrite ^/ http://localhost:9999/error.html
            break;
        }
    }
}
```

## gzip
```conf
server {
    gzip on;
    # 指定需要压缩的类型
    gzip_types application/javascript;
    # 静态压缩，在资源目录下已经有压缩好的.gz文件时进行使用
    gzip_static on;
    # 最小压缩大小，小于1k压缩意义不大
    gzip_min_length 1k;
    # 压缩级别，数字越大压缩效果越好，但使用的cpu越多
    gzip_comp_level 6;
    # 是否在响应头中设置very:accpet-encoding
    gzip_very on;
    # 设置压缩需要的缓冲区大小
    gzip_buffers 2 4k;
    # 对指定浏览器禁用gzip
    gzip_disable MSIE [1-6]\;
    # 启用gzip的最低http版本，默认1.1
    gzip_http_version 1.1;
}
```