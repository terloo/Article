# Oauth2
用于第三方向服务提供商申请用户相关授权(并获取用户相关资源)的验证协议

## 角色
1. Resource Owner：资源所有者，即用户
2. Third-party application：第三方应用，即需要获取用户授权的应用
3. Http Service：oauth2服务提供商
4. Authorization Server：授权服务器，即服务提供商处理授权相关逻辑的服务器
5. Resource Server：资源服务器，保存了用户相关资源
6. User Agent：用户代理，比如流浪其

## 授权模式
1. authorization_code：授权码模式，更为严格
2. implicit：简单模式
3. password：用户直接将账号密码交给第三方应用
4. client_credentials：用户以自己的账号密码来申请资源

## 授权码模式
1. 用户通过代理(浏览器)访问第三方客户端
2. 第三方客户端将用户导向授权服务器
   1. `https://授权服务器地址/oauth/authorization?client_id=xxx&response_type=code&scope=all&redirect_uri=http://第三方服务地址/重定向路由`
   2. client_id标识第三方应用，需要在授权服务器注册
   3. response_type为code，表示返回授权码
   4. scope：规定权限范围
   5. redirect_uri：授权成功后的重定向地址
3. 授权服务器(要求用户登录)询问用户是否授权
4. 用户同意授权
5. 授权服务器将用户导向第三方客户端指定的“重定向uri”，并附上授权码
   1. `https://第三方服务地址/重定向路由?authorization_code=xxxxxx`
6. 第三方客户端使用授权码向授权服务器申请令牌
7. 授权服务器核对令牌，向第三方客户端发送访问令牌(access token，可以使用jwt签名)和刷新令牌(refresh token)
   1. `https://授权服务器地址/oauth/token?client_id=xxx&client_secret=xxxx&grant_type=authorization_code&code=xxxxxx&redirect_uri=http://第三方服务地址/重定向路由`
   2. grant_type：授权模式
   3. code：在授权码模式时需要提供
8. 第三方客户端使用访问令牌向资源服务器请求资源

## 简单模式
1. 用户通过代理(浏览器)访问第三方客户端
2. 第三方客户端将用户导向授权服务器
   1. `https://授权服务器地址/oauth/authorization?client_id=xxx&response_type=token&scope=all&redirect_uri=http://第三方服务地址/重定向路由`
   2. client_id标识第三方应用，需要在授权服务器注册
   3. response_type为token，表示直接返回token
   4. scope：规定权限范围
   5. redirect_uri：授权成功后的重定向地址
3. 授权服务器(要求用户登录)询问用户是否授权
4. 用户同意授权
5. 第三方客户端使用访问令牌向资源服务器请求资源

## 密码模式
1. 用户通过代理(浏览器)访问第三方客户端
2. 第三方客户端直接要求用户输入账号密码
3. 第三方客户端使用用户账号密码访问授权服务器
   1. `https://授权服务器地址/oauth/token?client_id=xxx&client_secret=xxxx&grant_type=password&username=xxxxxx&password=xxxxxx&redirect_uri=http://第三方服务地址/重定向路由`
   2. grant_type为password
   3. username，password：用户的账号密码

## 客户端模式
1. 客户端以自己的名义向授权服务器请求授权
   1. `https://授权服务器地址/oauth/token?client_id=xxx&client_secret=xxxx&grant_type=client_credentials`