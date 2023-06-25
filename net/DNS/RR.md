# RR
DNS分布式数据库中保存的资源记录(Resource Record)

## 格式
`(name, value, type, ttl)`
1. name：记录名
2. value：记录值
3. ttl：time to live 记录的缓存时间
4. type：记录的类型
   1. A：name为主机，value为ip地址
   2. CNAME：name为规范名字的别名，value为规范名字
   3. NS：name为子域名，value为该子域名的权威服务器的域名
   4. MX：value为name对应的邮件服务器的名字