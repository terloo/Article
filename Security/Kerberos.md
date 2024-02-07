# Kerberos
Kerberos，域认证系统，在一个域中为用户提供安全的身份认证

## 域认证流程
1. client向Kerberos服务请求访问server的权限
   1. Kerberos得到了这个消息
   2. 首先判断client是否是可信赖的，即通过AS返回的黑白名单进行判断
   3. 成功后，返回TGT和AS给client
2. client得到了TGT后，带着TGT继续向Kerberos请求，希望获取访问server的权限
   1. Kerberos得到了这个消息
   2. 通过TGT判断出了client拥有访问server的权限，给了client访问server的权限ticket
3. client得到ticket后，带着ticket访问server，该ticket只对该server有效