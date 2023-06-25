# openAPI

## 获取k8s opoenAPI文档
1. `curl http://<masterIP>:<masterPort>/openapi/v2 > k8s-swagger.json`
2. 由于文档是一个很大的swagger格式的json文件，所以需要启动一个swagger ui服务来导入文档
3. `docker run -d -p 38888:8080 -e SWAGGER_JSON=/k8s-swagger.json -v /root/k8s-swagger.json:/k8s-swagger.json swaggerapi/swagger-ui`
4. 在ui中搜索`/k8s-swagger.json`