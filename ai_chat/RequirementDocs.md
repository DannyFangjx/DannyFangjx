## 需求文档

zapp

## 需求内容

1. 做一款ai应用。类似与 character.ai.

2. 用户登录进去之后，可以跟ai角色聊天。

3. 本项目中，所有http请求的应答均是如下格式，data为各个接口返回的数据，可能为nil。 err_code=0表示成功。
err_code不为0时，弹窗展示err_msg。
```
resp:
{
	"err_code": 0,
	"err_msg": "",
	"err_detail": "",
	"data": obj
}

```

## 需求详情

### 1.用户登录。

访问应用后，进入到首页。 首页布局如下图

![image.png](attachment:0d27461c-864a-4684-91e6-4305d5f226dc:image.png)

1. 用户输入邮箱/密码 登录

      背景图片就写死一张图即可。

      下面的，关于/xxxx 隐私条款/xxx 等等，都帮我生成一些即可。

### 2.登录后首页

首页布局如下所示:

![image.png](attachment:3998c8bc-b421-430e-a79a-cf48ecac42cb:image.png)

1. 聊天: 点击后会进入聊天页面。用户可以在此页面跟ai聊天。默认ai的id，aiId=10003
未来左边列表里，能跟哪些ai聊天，后续会调用服务端接口来获取ai列表。 然后跟列表里的ai聊天。
上述逻辑本期先不做，先固定跟10003聊天。

用户输入内容后，请求后端接口 POST api/zapp/chat/text_chat

## 后端文档

---
注意: 用户注册先不做。



1. 用户登录

```bash
req:
curl --location 'http://localhost:19998/api/zapp/auth/login' \
--header 'Content-Type: application/json' \
--data-raw '{
    "email":"383319845@qq.com",
    "password": "123123*"
}'

resp:
{
    "err_code": 0,
    "err_msg": "",
    "err_detail": "",
    "data": {
        "expires": 86400000,
        "refresh_token": "kRheJWc4OUx0G0O9Un4S3RJ6hlGROgAORXN-YhuXnNIsLfxdsgWdqkX2D_Ut_q2I",
        "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjE4OWEwMWM5LTk3MjItNDhhZC1iYjc2LTMzZGIwYjE3NmZkNyIsInJvbGUiOiIxMzkyYTU4My0yNjQ4LTQ2M2UtODJkMy00MjU2YjY1OGMzZTEiLCJhcHBfYWNjZXNzIjpmYWxzZSwiYWRtaW5fYWNjZXNzIjpmYWxzZSwiaWF0IjoxNzQ5ODExNDk4LCJleHAiOjE3NDk4OTc4OTgsImlzcyI6ImRpcmVjdHVzIn0.MeKBR7g8IK0IplX01hyKt9tJ9Zw0gS-XL0FxfwMKF3w"
    }
}

expires 单位ms，是指access_token的过期时间。 

```
2. 登出
```
curl --location 'http://localhost:19998/api/zapp/auth/logout' \
--header 'Content-Type: application/json' \
--data '{
    "refresh_token":"iQsz9XzQyrDtofwf-VA94vmqeniVUv6qZQPFKm1tAhBB1knRAqn43eusTirRIh06"
}'


{
    "err_code": 0,
    "err_msg": "",
    "err_detail": "",
    "data": {}
}

```

3. 刷新token
```
curl --location 'http://localhost:19998/api/zapp/auth/refresh' \
--header 'Content-Type: application/json' \
--data '{
    "refresh_token":"cPX6sSEUN0ZZyCOg0vnKYCBdRzeCy1d6Ucx-KZFmp1ZuXhLT4CX9BTjZ4TAlaobM"
}'

{
    "err_code": 200102,
    "err_msg": "error_server",
    "err_detail": "refresh failed",
    "data": {}
}


```

4. 获取用户信息
```
curl --location --request GET 'http://localhost:19998/api/zapp/user/get_info' \
--header 'token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjE4OWEwMWM5LTk3MjItNDhhZC1iYjc2LTMzZGIwYjE3NmZkNyIsInJvbGUiOiIxMzkyYTU4My0yNjQ4LTQ2M2UtODJkMy00MjU2YjY1OGMzZTEiLCJhcHBfYWNjZXNzIjpmYWxzZSwiYWRtaW5fYWNjZXNzIjpmYWxzZSwiaWF0IjoxNzQ5ODExNDk4LCJleHAiOjE3NDk4OTc4OTgsImlzcyI6ImRpcmVjdHVzIn0.MeKBR7g8IK0IplX01hyKt9tJ9Zw0gS-XL0FxfwMKF3w' \
--header 'Content-Type: text/plain' \
--header 'Authorization: ••••••' \
--data-raw '{
    "email":"383319845@qq.com",
    "password": "123123123*"
}'


{
    "err_code": 0,
    "err_msg": "",
    "err_detail": "",
    "data": {
        "item": {
            "uid": "189a01c9-9722-48ad-bb76-33db0b176fd7",
            "first_name": "dannyFang",
            "email": "383319845@qq.com"
        }
    }
}
```

5. 更新用户信息

目前只支持更新first_name和 password 。 
传入的非空，则更新，否则不更新

```
curl --location 'http://localhost:19998/api/zapp/user/update' \
--header 'token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjE4OWEwMWM5LTk3MjItNDhhZC1iYjc2LTMzZGIwYjE3NmZkNyIsInJvbGUiOiIxMzkyYTU4My0yNjQ4LTQ2M2UtODJkMy00MjU2YjY1OGMzZTEiLCJhcHBfYWNjZXNzIjpmYWxzZSwiYWRtaW5fYWNjZXNzIjpmYWxzZSwiaWF0IjoxNzQ5ODExNDk4LCJleHAiOjE3NDk4OTc4OTgsImlzcyI6ImRpcmVjdHVzIn0.MeKBR7g8IK0IplX01hyKt9tJ9Zw0gS-XL0FxfwMKF3w' \
--header 'Content-Type: application/json' \
--header 'Authorization: ••••••' \
--data '{
    "password": "12345678",
    "first_name":"dannyFang"
}'

{
    "err_code": 0,
    "err_msg": "",
    "err_detail": "",
    "data": {}
}


```




6. 用户跟ai聊天
    
     POST api/zapp/ai/chat/text_chat
    
    ```json
    传入参数: 
    
    httpHeader 
    access_token: "eyJhbGciOiJI..."
    
    req:
    {
    aiId: 10003,
    content:"你是谁"
    }
    
    resp:
    {
    "err_code": 0,
    "err_msg": "",
    "err_detail": "",
    "data": {
    	"content": "我在吃饭"
    	}
    }
    
    ```