# CRI
CRI使用grpc和protocol buffer技术，将对container runtime的调用转换为远程调用。使用kubelet作为grpc的客户端，container runtime作为服务端。  
CRI规定了远程调用的接口，container runtime只要实现该接口即可。