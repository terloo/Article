# 删除Pod流程

更新DeletionGracePeriodSeconds：由 30 到 30
needUpdate： false needReconcile： false needGracefulDelete： true
主循环  2022-06-30 19:52:40.992660188 +0800 CST m=+137.985686564 configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resousion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","deletionTimestamp":"2022-06-30T11:53:10Z","deletionGracePeriodSeconds":30,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-30T11:52:25Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":true}],"qosClass":"BestEffort"}}],"Op":2,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resourceVersion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","delnTimestamp":"2022-06-30T11:53:10Z","deletionGracePeriodSeconds":30,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-30T11:52:25Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":true}],"qosClass":"BestEffort"}}
2022-06-30 19:52:40.993929173 +0800 CST m=+137.986955550 执行syncTerminating，当前container状态： {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Name":"busybox2","Namespace":"default","IPs":["10"],"ContainerStatuses":[{"ID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","Name":"busybox","State":"running","CreatedAt":"2022-06-30T19:52:25.459655323+08:00","StartedAt":"2022-06-30T19:52:25.540634836+08:00","FinishedAt":"0001-01-01T00:00:00Z","ExitCode":0,"Image":"busybox:latest","ImageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","Hash":774174649,"RestartCount":0,"Reason":"","Message":""}],"SandboxStatuses":[{"id":"10ab4341cc220ece04ddcbb388bbcf81570ca3fa836d4f38a704a76ba6363b2f","metadata":{"name":"busybox2","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","namespace":"default"},"created_at":1656589942088222432,"network":{"ip":"10.10.1.46"},"linux":{"namespaces":{"options":{"pid":1}}},"labels":{"app":"busybox","io.kubernetes.pod.name":"busybox2","io.kubernetes.pod.namespace":"default","io.kubernetes.pod.uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"}}]}
2022-06-30 19:52:43.166287649 +0800 CST m=+140.159314025 设置StatusManager中Pod状态： {"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionnull},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":null}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-30T11:52:25Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":true}],"qosClass":"BestEffort"}
2022-06-30 19:52:43.168553051 +0800 CST m=+140.161579421 Kubelet.killPod
StatusManager循环  2022-06-30 19:52:43.192122054 +0800 CST m=+140.185148431 同步状态到api-server
2022-06-30 19:53:13.315611673 +0800 CST m=+170.308638043 Kubelet.killPod完毕
2022-06-30 19:53:13.320212866 +0800 CST m=+170.313239248 处理完一个PodWork，但发现有新的任务在排队 {"WorkType":2,"Options":{"UpdateType":0,"StartTime":"0001-01-01T00:00:00Z","Pod":{"metadata":{,"namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resourceVersion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","deletionTimestamp":"2022-06-30T11:53:10Z","deletionGracePeriodSeconds":30,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-30T11:52:25Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":true}],"qosClass":"BestEffort"}},"MirrorPod":null,"RunningPod":null,"KillPodOptions":null}}
主循环  2022-06-30 19:53:13.373931529 +0800 CST m=+170.366957895 pleg的消息 {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Type":"ContainerDied","Data":"93ce7bf040dc8f0d821684ddfa70857b07759f1361a19517d98cf76fff"}
主循环  2022-06-30 19:53:13.374442545 +0800 CST m=+170.367468913 pleg的消息 {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Type":"ContainerDied","Data":"10ab4341cc220ece04ddcbb388bbcf81570ca3fa88a704a76ba6363b2f"}
2022-06-30 19:53:13.374383202 +0800 CST m=+170.367409583 执行syncTerminatedPod，当前container状态： {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Name":"busybox2","Namespace":"default","IPs":["46"],"ContainerStatuses":[{"ID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","Name":"busybox","State":"exited","CreatedAt":"2022-06-30T19:52:25.459655323+08:00","StartedAt":"2022-06-30T19:52:25.540634836+08:00","FinishedAt":"2022-06-30T19:53:13.23192747+08:00","ExitCode":137,"Image":"busybox:latest","ImageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","Hash":774174649,"RestartCount":0,"Reason":"Error","Message":""}],"SandboxStatuses":[{"id":"10ab4341cc220ece04ddcbb388bbcf81570ca3fa836d4f38a704a76ba6363b2f","metadata":{"name":"busybox2","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","namespace":"default"},"state":1,"created_at":1656589942088222432,"network":{},"linux":{"namespaces":{"options":{"pid":1}}},"labels":{"app":"busybox","io.kubernetes.pod.name":"busybox2","io.kubernetes.pod.namespace":"default","io.kubernetes.pod.uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"}}]}
2022-06-30 19:53:13.375405722 +0800 CST m=+170.368432100 设置StatusManager中Pod状态： {"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionnull},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":null,"reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":null,"reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":null}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}
E0630 19:53:13.383765  105945 remote_runtime.go:572] "ContainerStatus from runtime service failed" err="rpc error: code = Unknown desc = Error: No such container: 93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff" containerID="93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"
StatusManager循环  2022-06-30 19:53:13.390762788 +0800 CST m=+170.383789164 同步状态到api-server
needUpdate： false needReconcile： true needGracefulDelete： false
主循环  2022-06-30 19:53:13.412117445 +0800 CST m=+170.405143825 configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resousion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","deletionTimestamp":"2022-06-30T11:53:10Z","deletionGracePeriodSeconds":30,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}}],"Op":5,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resourceVersion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","delnTimestamp":"2022-06-30T11:53:10Z","deletionGracePeriodSeconds":30,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}}
StatusManager循环  2022-06-30 19:53:13.408986095 +0800 CST m=+170.402012471 同步状态到api-server
StatusManager循环  2022-06-30 19:53:13.413674457 +0800 CST m=+170.406700833 Pod应该被删除。。。。。。。。。
更新DeletionGracePeriodSeconds：由 0 到 0
needUpdate： false needReconcile： false needGracefulDelete： true
主循环  2022-06-30 19:53:13.418224823 +0800 CST m=+170.411251200 configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resousion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","deletionTimestamp":"2022-06-30T11:52:40Z","deletionGracePeriodSeconds":0,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}}],"Op":2,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resourceVersion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","delnTimestamp":"2022-06-30T11:52:40Z","deletionGracePeriodSeconds":0,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}}
主循环  2022-06-30 19:53:13.42055819 +0800 CST m=+170.413584572 configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","resourion":"91426","creationTimestamp":"2022-06-30T11:52:21Z","deletionTimestamp":"2022-06-30T11:52:40Z","deletionGracePeriodSeconds":0,"labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-30T19:52:21.770103945+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-30T11:52:21Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-vps8f","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-vps8f","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:53:13Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"terminated":{"exitCode":137,"reason":"Error","startedAt":"2022-06-30T11:52:25Z","finishedAt":"2022-06-30T11:53:13Z","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":false}],"qosClass":"BestEffort"}}],"Op":3,"Source":"api"}
podManager删除Pod缓存
2022-06-30 19:53:13.679999245 +0800 CST m=+170.673025621 终止StatusManager中Pod状态： {"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransition"2022-06-30T11:52:21Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:26Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-30T11:52:21Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.46","podIPs":[{"ip":"10.10.1.46"}],"startTime":"2022-06-30T11:52:21Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-30T11:52:25Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff","started":true}],"qosClass":"BestEffort"}
主循环  2022-06-30 19:53:14.378900355 +0800 CST m=+171.371926728 pleg的消息 {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Type":"ContainerRemoved","Data":"93ce7bf040dc8f0d821684ddfa70857b07759f3721a19517d98cf76fff"}
主循环  2022-06-30 19:53:24.449019688 +0800 CST m=+181.442046120 pleg的消息 {"ID":"4bd36c8b-cfa7-4b04-b260-5b9966f0e96e","Type":"ContainerRemoved","Data":"10ab4341cc220ece04ddcbb388bbcf81570ca34f38a704a76ba6363b2f"}

1. api-server收到删除请求，将资源metadata.deletionTimestamp和metadata.deletionGracePeriodSeconds赋值
   ```json
   {
      "Pods": [
         {
               "metadata": {
                  "name": "busybox2",
                  "namespace": "default",
                  "uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
                  "resousion": "91426",
                  "creationTimestamp": "2022-06-30T11:52:21Z",
                  "deletionTimestamp": "2022-06-30T11:53:10Z",
                  "deletionGracePeriodSeconds": 30,
                  "labels": {
                     "app": "busybox"
                  },
                  "annotations": {
                     "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                     "kubernetes.io/config.seen": "2022-06-30T19:52:21.770103945+08:00",
                     "kubernetes.io/config.source": "api"
                  },
               },
               "spec": {
                  "volumes": [
                     {
                           "name": "kube-api-access-vps8f",
                           "projected": {
                              "sources": [
                                 {
                                       "serviceAccountToken": {
                                          "expirationSeconds": 3607,
                                          "path": "token"
                                       }
                                 },
                                 {
                                       "configMap": {
                                          "name": "kube-root-ca.crt",
                                          "items": [
                                             {
                                                   "key": "ca.crt",
                                                   "path": "ca.crt"
                                             }
                                          ]
                                       }
                                 },
                                 {
                                       "downwardAPI": {
                                          "items": [
                                             {
                                                   "path": "namespace",
                                                   "fieldRef": {
                                                      "apiVersion": "v1",
                                                      "fieldPath": "metadata.namespace"
                                                   }
                                             }
                                          ]
                                       }
                                 }
                              ],
                              "defaultMode": 420
                           }
                     }
                  ],
                  "containers": [
                     {
                           "name": "busybox",
                           "image": "busybox",
                           "command": [
                              "/bin/sh",
                              "-c",
                              "sleep 36000;"
                           ],
                           "resources": {},
                           "volumeMounts": [
                              {
                                 "name": "kube-api-access-vps8f",
                                 "readOnly": true,
                                 "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
                              }
                           ],
                           "terminationMessagePath": "/dev/termination-log",
                           "terminationMessagePolicy": "File",
                           "imagePullPolicy": "Always"
                     }
                  ],
                  "restartPolicy": "Always",
                  "terminationGracePeriodSeconds": 30,
                  "dnsPolicy": "ClusterFirst",
                  "serviceAccountName": "default",
                  "serviceAccount": "default",
                  "nodeName": "node1",
                  "securityContext": {},
                  "schedulerName": "default-scheduler",
                  "tolerations": [
                     {
                           "key": "node.kubernetes.io/not-ready",
                           "operator": "Exists",
                           "effect": "NoExecute",
                           "tolerationSeconds": 300
                     },
                     {
                           "key": "node.kubernetes.io/unreachable",
                           "operator": "Exists",
                           "effect": "NoExecute",
                           "tolerationSeconds": 300
                     }
                  ],
                  "priority": 0,
                  "enableServiceLinks": true,
                  "preemptionPolicy": "PreemptLowerPriority"
               },
               "status": {
                  "phase": "Running",
                  "conditions": [
                     {
                           "type": "Initialized",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:21Z"
                     },
                     {
                           "type": "Ready",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:26Z"
                     },
                     {
                           "type": "ContainersReady",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:26Z"
                     },
                     {
                           "type": "PodScheduled",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:21Z"
                     }
                  ],
                  "hostIP": "192.168.100.101",
                  "podIP": "10.10.1.46",
                  "podIPs": [
                     {
                           "ip": "10.10.1.46"
                     }
                  ],
                  "startTime": "2022-06-30T11:52:21Z",
                  "containerStatuses": [
                     {
                           "name": "busybox",
                           "state": {
                              "running": {
                                 "startedAt": "2022-06-30T11:52:25Z"
                              }
                           },
                           "lastState": {},
                           "ready": true,
                           "restartCount": 0,
                           "image": "busybox:latest",
                           "imageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
                           "containerID": "docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff",
                           "started": true
                     }
                  ],
                  "qosClass": "BestEffort"
               }
         }
      ],
      "Op": 2,
      "Source": "api"
   }
   ```
2. 主循环收到configCh消息，消息类型为DELETE(Pod状态未改变，且metadata.deletionTimestamp不为空)
   ```json
   {
      "ID": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
      "Name": "busybox2",
      "Namespace": "default",
      "IPs": [
         "10"
      ],
      "ContainerStatuses": [
         {
               "ID": "docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff",
               "Name": "busybox",
               "State": "running",
               "CreatedAt": "2022-06-30T19:52:25.459655323+08:00",
               "StartedAt": "2022-06-30T19:52:25.540634836+08:00",
               "FinishedAt": "0001-01-01T00:00:00Z",
               "ExitCode": 0,
               "Image": "busybox:latest",
               "ImageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
               "Hash": 774174649,
               "RestartCount": 0,
               "Reason": "",
               "Message": ""
         }
      ],
      "SandboxStatuses": [
         {
               "id": "10ab4341cc220ece04ddcbb388bbcf81570ca3fa836d4f38a704a76ba6363b2f",
               "metadata": {
                  "name": "busybox2",
                  "uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
                  "namespace": "default"
               },
               "created_at": 1656589942088222432,
               "network": {
                  "ip": "10.10.1.46"
               },
               "linux": {
                  "namespaces": {
                     "options": {
                           "pid": 1
                     }
                  }
               },
               "labels": {
                  "app": "busybox",
                  "io.kubernetes.pod.name": "busybox2",
                  "io.kubernetes.pod.namespace": "default",
                  "io.kubernetes.pod.uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e"
               },
               "annotations": {
                  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                  "kubernetes.io/config.seen": "2022-06-30T19:52:21.770103945+08:00",
                  "kubernetes.io/config.source": "api"
               }
         }
      ]
   }
   ```
3. 更新PodManager缓存，交由PodWorkes处理
4. PodWorkers从PodCache中获取kubecontainer.PodStatus(此时容器仍为running状态)并计算出优雅删除时间，调用Kubelet.syncTerminatingPod将其传入
   ```json
   {
      "ID": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
      "Name": "busybox2",
      "Namespace": "default",
      "IPs": [
         "10"
      ],
      "ContainerStatuses": [
         {
               "ID": "docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff",
               "Name": "busybox",
               "State": "running",
               "CreatedAt": "2022-06-30T19:52:25.459655323+08:00",
               "StartedAt": "2022-06-30T19:52:25.540634836+08:00",
               "FinishedAt": "0001-01-01T00:00:00Z",
               "ExitCode": 0,
               "Image": "busybox:latest",
               "ImageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
               "Hash": 774174649,
               "RestartCount": 0,
               "Reason": "",
               "Message": ""
         }
      ],
      "SandboxStatuses": [
         {
               "id": "10ab4341cc220ece04ddcbb388bbcf81570ca3fa836d4f38a704a76ba6363b2f",
               "metadata": {
                  "name": "busybox2",
                  "uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
                  "namespace": "default"
               },
               "created_at": 1656589942088222432,
               "network": {
                  "ip": "10.10.1.46"
               },
               "linux": {
                  "namespaces": {
                     "options": {
                           "pid": 1
                     }
                  }
               },
               "labels": {
                  "app": "busybox",
                  "io.kubernetes.pod.name": "busybox2",
                  "io.kubernetes.pod.namespace": "default",
                  "io.kubernetes.pod.uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e"
               },
               "annotations": {
                  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                  "kubernetes.io/config.seen": "2022-06-30T19:52:21.770103945+08:00",
                  "kubernetes.io/config.source": "api"
               }
         }
      ]
   }
   ```
5. Kubelet.syncTerminatingPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，容器状态仍是running状态，但会强制同步至api-server。产生一个异步流程
   1. StatusManager将状态同步至api-server
   2. api来源收到消息，由于状态未发生变化。不向发送configCh消息主循环
6. 调用Kubelet.killPod，此方法会阻塞持续metadata.deletionGracePeriodSeconds的时间。期间PLEG会推送消息，被PodWorkers暂存
   ```json
   {
      "ID": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
      "Type": "ContainerDied",
      "Data": "93ce7bf040dc8f0d821684ddfa70857b07759f1361a19517d98cf76fff"
   }
   ```
7. 执行完Kubelet.killPod，PodWorker会在后处理中向lastUndeliveredWorkUpdate中写入一个WorkType为Terminated的消息(由于优雅退出时间较长，会覆盖掉期间产生的所有暂存消息)
   ```json
   {
      "WorkType": 2,
      "Options": {
         "UpdateType": 0,
         "StartTime": "0001-01-01T00:00:00Z",
         "Pod": {
               "metadata": {
                  "namespace": "default",
                  "uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
                  "resourceVersion": "91426",
                  "creationTimestamp": "2022-06-30T11:52:21Z",
                  "deletionTimestamp": "2022-06-30T11:53:10Z",
                  "deletionGracePeriodSeconds": 30,
                  "labels": {
                     "app": "busybox"
                  },
                  "annotations": {
                     "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                     "kubernetes.io/config.seen": "2022-06-30T19:52:21.770103945+08:00",
                     "kubernetes.io/config.source": "api"
                  },
               },
               "spec": {
                  "volumes": [
                     {
                           "name": "kube-api-access-vps8f",
                           "projected": {
                              "sources": [
                                 {
                                       "serviceAccountToken": {
                                          "expirationSeconds": 3607,
                                          "path": "token"
                                       }
                                 },
                                 {
                                       "configMap": {
                                          "name": "kube-root-ca.crt",
                                          "items": [
                                             {
                                                   "key": "ca.crt",
                                                   "path": "ca.crt"
                                             }
                                          ]
                                       }
                                 },
                                 {
                                       "downwardAPI": {
                                          "items": [
                                             {
                                                   "path": "namespace",
                                                   "fieldRef": {
                                                      "apiVersion": "v1",
                                                      "fieldPath": "metadata.namespace"
                                                   }
                                             }
                                          ]
                                       }
                                 }
                              ],
                              "defaultMode": 420
                           }
                     }
                  ],
                  "containers": [
                     {
                           "name": "busybox",
                           "image": "busybox",
                           "command": [
                              "/bin/sh",
                              "-c",
                              "sleep 36000;"
                           ],
                           "resources": {},
                           "volumeMounts": [
                              {
                                 "name": "kube-api-access-vps8f",
                                 "readOnly": true,
                                 "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
                              }
                           ],
                           "terminationMessagePath": "/dev/termination-log",
                           "terminationMessagePolicy": "File",
                           "imagePullPolicy": "Always"
                     }
                  ],
                  "restartPolicy": "Always",
                  "terminationGracePeriodSeconds": 30,
                  "dnsPolicy": "ClusterFirst",
                  "serviceAccountName": "default",
                  "serviceAccount": "default",
                  "nodeName": "node1",
                  "securityContext": {},
                  "schedulerName": "default-scheduler",
                  "tolerations": [
                     {
                           "key": "node.kubernetes.io/not-ready",
                           "operator": "Exists",
                           "effect": "NoExecute",
                           "tolerationSeconds": 300
                     },
                     {
                           "key": "node.kubernetes.io/unreachable",
                           "operator": "Exists",
                           "effect": "NoExecute",
                           "tolerationSeconds": 300
                     }
                  ],
                  "priority": 0,
                  "enableServiceLinks": true,
                  "preemptionPolicy": "PreemptLowerPriority"
               },
               "status": {
                  "phase": "Running",
                  "conditions": [
                     {
                           "type": "Initialized",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:21Z"
                     },
                     {
                           "type": "Ready",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:26Z"
                     },
                     {
                           "type": "ContainersReady",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:26Z"
                     },
                     {
                           "type": "PodScheduled",
                           "status": "True",
                           "lastProbeTime": null,
                           "lastTransitionTime": "2022-06-30T11:52:21Z"
                     }
                  ],
                  "hostIP": "192.168.100.101",
                  "podIP": "10.10.1.46",
                  "podIPs": [
                     {
                           "ip": "10.10.1.46"
                     }
                  ],
                  "startTime": "2022-06-30T11:52:21Z",
                  "containerStatuses": [
                     {
                           "name": "busybox",
                           "state": {
                              "running": {
                                 "startedAt": "2022-06-30T11:52:25Z"
                              }
                           },
                           "lastState": {},
                           "ready": true,
                           "restartCount": 0,
                           "image": "busybox:latest",
                           "imageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
                           "containerID": "docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff",
                           "started": true
                     }
                  ],
                  "qosClass": "BestEffort"
               }
         },
         "MirrorPod": null,
         "RunningPod": null,
         "KillPodOptions": null
      }
   }
   ```
8. PodWoker开始处理Terminated消息，从PodCache中获取kubecontainer.PodStatus(此时容器已为exited状态)，调用Kubelet.syncTerminatedPod将其传入
   ```json
   {
      "ID": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
      "Name": "busybox2",
      "Namespace": "default",
      "IPs": [
         "46"
      ],
      "ContainerStatuses": [
         {
               "ID": "docker://93ce7bf040dc8f0d821684ddfa70857b07759f136c023721a19517d98cf76fff",
               "Name": "busybox",
               "State": "exited",
               "CreatedAt": "2022-06-30T19:52:25.459655323+08:00",
               "StartedAt": "2022-06-30T19:52:25.540634836+08:00",
               "FinishedAt": "2022-06-30T19:53:13.23192747+08:00",
               "ExitCode": 137,
               "Image": "busybox:latest",
               "ImageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
               "Hash": 774174649,
               "RestartCount": 0,
               "Reason": "Error",
               "Message": ""
         }
      ],
      "SandboxStatuses": [
         {
               "id": "10ab4341cc220ece04ddcbb388bbcf81570ca3fa836d4f38a704a76ba6363b2f",
               "metadata": {
                  "name": "busybox2",
                  "uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e",
                  "namespace": "default"
               },
               "state": 1,
               "created_at": 1656589942088222432,
               "network": {},
               "linux": {
                  "namespaces": {
                     "options": {
                           "pid": 1
                     }
                  }
               },
               "labels": {
                  "app": "busybox",
                  "io.kubernetes.pod.name": "busybox2",
                  "io.kubernetes.pod.namespace": "default",
                  "io.kubernetes.pod.uid": "4bd36c8b-cfa7-4b04-b260-5b9966f0e96e"
               },
               "annotations": {
                  "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                  "kubernetes.io/config.seen": "2022-06-30T19:52:21.770103945+08:00",
                  "kubernetes.io/config.source": "api"
               }
         }
      ]
   }
   ```
9. Kubelet.syncTerminatingPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，容器状态为terminated，由于新旧状态不一致，开始同步。产生一个异步流程
   1.  StatusManager将状态同步至api-server
   2.  主循环收到configCh消息，消息类型为RECONCILE(这是由于本次同步只更新了Pod状态)
   3.  更新PodManager缓存，然后操作被PodWorkers暂存
10. Kubelet.syncTerminatingPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，容器状态为terminated。强制同步，产生异步流程。
    1.  StatusManager将状态同步至api-server
    2.  api来源收到消息，由于不是有意义改动且状态未发生变化。不向发送configCh消息主循环
11. Kubelet.syncTerminatingPod中调用StatusManager的TerminatePod方法。产生一个异步流程
    1. StatusManager发送metadata.deletionGracePeriodSeconds为0的delete请求到api-server
    2. api-server把metadata.deletionGracePeriodSeconds修改为0，然后直接删除ETCD中的资源对象
    3. 主循环收到两条configCh消息，消息类型为DELETE(metadata.deletionGracePeriodSeconds修改为0，且metadata.deletionTimestamp不为空)和REMOVE(ETCD中资源对象被真实移除)
    4. 更新PodManager缓存，然后操作被PodWorkers暂存
12. PodWorker进入后处理，直接清理缓存并退出PodWorker循环，暂存的所有工作全部忽略