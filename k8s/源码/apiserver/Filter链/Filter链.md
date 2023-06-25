# Filter链
k8s中用于处理资源相关请求的过滤器，Filter主要是添加和修改HTTP请求的header，添加和修改请求的Context

## 所有过滤器
1. WithAuditID：给资源添加AuditID
2. WithPanicRecovery：进行panic恢复
3. WithMuxAndDiscoveryCompelete：等待Mux和Discovery初始化完成
4. WithRequestInfo：向Context中添加请求相关的信息
5. WithLatencyTrackers
6. WithTracing：添加trace，能观测到更加详细的log
7. WithHTTPloggin
8. WithRetryAfter：在某些状态下(例如apiserver正在关闭)，通知请求者间隔一段时间后重试
9. WithHSTS
10. WithCacheControl：禁用HTTP本身的缓存功能
11. WithWarnningRecorder
12. WithAuditAnnotations：向Context中添加Audit使用的注解
13. WithProbabilisticGoaway：在某些条件下，拒绝该次请求。一般用于负载均衡
14. WithWaitGroup：全局waitGroup加一，在apiserver关闭时会等待waitGroup的释放
15. WithRequestDeadline：请求超时时间
16. WithTimeoutForNonLongRunningRequests
17. WithCORS：跨域处理
18. WithAuthentication：认证
19. WithAudit：审计日志记录
20. WithImpersonation：替换当前User信息
21. WithMaxInFlightLimit：流量控制
22. WithAuthorization：鉴权