---
layout: post
title: spring mvc 返回图片
date: 2018-03-12 11:16:00
categories:
    - 日常记录
tags:
    - spring mvc
    - spring
---
# spring mvc 返回图片

## 说明

```java
@RestController

...

    @PostMapping("wxa/code")
    public ResponseEntity<?> getWxaShareCode() {
        try {

            ByteArrayOutputStream byteArrayOutputStream =
                    TucHttpUtil.httpPostInputStream("https://developers.weixin.qq.com/miniprogram/dev/image/qrcode/qrcode.png?t=18082721", null);

            InputStream is = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
            return ResponseEntity.ok()
                    .header("content-type","image/png")
                    .body(new InputStreamResource(is));

        } catch (Exception e) {
            return ResponseEntity.notFound().build();
        }
    }
...
```

1. 使用ResponseEntity可以和@RestController共用
2. ResponseEntity是restful的响应体, 相当于sevlet的HttpSevletResponse的设置
3. InputSteam如果直接返回可能会导致流无法关闭, 使用ByteArrayOutputStream把数据保存在内存, 然后返回(可能会涉及回收问题)
4. 返回文件(比如图片), 需要设置header("content-type","image/png")