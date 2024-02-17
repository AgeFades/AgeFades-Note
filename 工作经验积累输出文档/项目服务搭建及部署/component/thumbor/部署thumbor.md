[TOC]

# 部署thumbor

[参考链接](https://blog.xiaoz.org/archives/17804)

## docker-compose.yaml

```shell
vim docker-compose-thumbor.yaml
```

```apl
version: '3.3'
services:
    thumbor:
        ports:
            - '18888:80'
        restart: always
        container_name: thumbor
        volumes:
            - './thumbor.conf:/app/thumbor.conf'
            - './result_storage:/tmp/thumbor/result_storage'
            - './storage:/tmp/thumbor/storage'
        image: minimalcompact/thumbor
```

## 配置文件

```bash
vim thumbor.conf
```

```apl
#裁剪最大宽度，意味着裁剪的图片不会超过此像素，默认是0（不限制）
MAX_WIDTH = 1200
#裁剪最大高度，意味着裁剪的图片不会超过此像素，默认是0（不限制）
MAX_HEIGHT = 800
#裁剪最小宽度，意味着裁剪的图片不会低于此像素，默认是1
MIN_WIDTH = 50
#裁剪最小高度，意味着裁剪的图片不会低于此像素，默认是1
MIN_HEIGHT = 50
#允许裁剪的图片的域名，支持正则表达式，支持多个图片
ALLOWED_SOURCES = ['.+\.abc\.com']
#裁剪图像的质量，默认为80
QUALITY = 95
#图像应保留在浏览器缓存中的秒数。它与 Expires 和 Cache-Control 标头直接相关
MAX_AGE = 7 * 24 * 60 * 60
#当图像在检测中出现错误或延迟排队时，为图像缓存设置一个低得多的过期时间会很方便。这样浏览器将更快地请求正确的图像。
MAX_AGE_TEMP_IMAGE = 60
#裁剪后的图像保存到哪个引擎，下方指的是保存到本地存储，从而进行缓存
RESULT_STORAGE = 'thumbor.result_storages.file_storage'
#原始图像保存时间
STORAGE_EXPIRATION_SECONDS = 864000
#裁剪后的图像保存时间
RESULT_STORAGE_EXPIRATION_SECONDS = 864000
#裁剪后的图片保存到哪个位置
RESULT_STORAGE_FILE_STORAGE_ROOT_PATH = '/tmp/thumbor/result_storage'
#开启结果缓存（裁剪图像的缓存）
RESULT_STORAGE_STORES_UNSAFE = True
```

## 简易部署脚本

```bash
vim deploy.sh
```

```bash
docker-compose -f docker-compose-thumbor.yaml down
docker-compose -f docker-compose-thumbor.yaml up -d
```

