configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lls":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","qosClass":"BestEffort"}}],"Op":1,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lab:{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","qosClass":"BestEffort"}}
2022-06-29 21:38:17.314662911 +0800 CST m=+98.885233445  执行syncPod，当前container状态： {"ID":"95bdac1a-7a17-40c9-b75d-e07816859760","Name":"","Namespace":"","IPs":null,"ContainerStatuses":nuboxStatuses":null}
2022-06-29 21:38:17.314990875 +0800 CST m=+98.885561403 设置Pod状态： {"phase":"Pending","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"tyReady","status":"False","lastProbeTime":null,"lastTransitionTime":null,"reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":null,"reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":null}],"hostIP":"192.168.100.101","containerStatuses":[{"name":"busybox","state":{"waiting":{"reason":"ContainerCreating"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox","imageID":"","started":false}],"qosClass":"BestEffort"}
同步状态到api-server
configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lls":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"waiting":{"reason":"ContainerCreating"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox","imageID":"","started":false}],"qosClass":"BestEffort"}}],"Op":5,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lab:{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"waiting":{"reason":"ContainerCreating"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox","imageID":"","started":false}],"qosClass":"BestEffort"}}
2022-06-29 21:38:17.624318853 +0800 CST m=+99.194889383 调用CRI SyncPod
pleg的消息 {"ID":"95bdac1a-7a17-40c9-b75d-e07816859760","Type":"ContainerStarted","Data":"3e758a520626620d85d010898ee1e1e4735fa1778d83886289e3ec776cfef1e5"}
2022-06-29 21:38:33.647503513 +0800 CST m=+115.218074042 调用CRI SyncPod完毕
处理完一个PodWork，但发现有新的任务在排队 {"WorkType":0,"Options":{"UpdateType":0,"StartTime":"2022-06-29T21:38:18.278893386+08:00","Pod":{"metadata":{"name":"busybox2","namespace":"default","u7-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"waiting":{"reason":"ContainerCreating"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox","imageID":"","started":false}],"qosClass":"BestEffort"}},"MirrorPod":null,"RunningPod":null,"KillPodOptions":null}}
pleg的消息 {"ID":"95bdac1a-7a17-40c9-b75d-e07816859760","Type":"ContainerStarted","Data":"c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4"}
2022-06-29 21:38:34.376737168 +0800 CST m=+115.947307698  执行syncPod，当前container状态： {"ID":"95bdac1a-7a17-40c9-b75d-e07816859760","Name":"busybox2","Namespace":"default","IPs":["10.10.1.3tainerStatuses":[{"ID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","Name":"busybox","State":"running","CreatedAt":"2022-06-29T21:38:33.566069909+08:00","StartedAt":"2022-06-29T21:38:33.644378832+08:00","FinishedAt":"0001-01-01T00:00:00Z","ExitCode":0,"Image":"busybox:latest","ImageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","Hash":534858999,"RestartCount":0,"Reason":"","Message":""}],"SandboxStatuses":[{"id":"3e758a520626620d85d010898ee1e1e4735fa1778d83886289e3ec776cfef1e5","metadata":{"name":"busybox2","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","namespace":"default"},"created_at":1656509897627665587,"network":{"ip":"10.10.1.31"},"linux":{"namespaces":{"options":{"pid":1}}},"labels":{"app":"busybox","io.kubernetes.pod.name":"busybox2","io.kubernetes.pod.namespace":"default","io.kubernetes.pod.uid":"95bdac1a-7a17-40c9-b75d-e07816859760"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"}}]}
2022-06-29 21:38:34.376993716 +0800 CST m=+115.947564248 设置Pod状态： {"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"t"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":null}],"hostIP":"192.168.100.101","podIP":"10.10.1.31","podIPs":[{"ip":"10.10.1.31"}],"containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-29T13:38:33Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","started":true}],"qosClass":"BestEffort"}
2022-06-29 21:38:34.377356467 +0800 CST m=+115.947926997 调用CRI SyncPod
2022-06-29 21:38:34.377591937 +0800 CST m=+115.948162467 调用CRI SyncPod完毕
处理完一个PodWork，但发现有新的任务在排队 {"WorkType":0,"Options":{"UpdateType":0,"StartTime":"2022-06-29T21:38:34.376674145+08:00","Pod":{"metadata":{"name":"busybox2","namespace":"default","u7-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","labels":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z","reason":"ContainersNotReady","message":"containers with unready status: [busybox]"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"waiting":{"reason":"ContainerCreating"}},"lastState":{},"ready":false,"restartCount":0,"image":"busybox","imageID":"","started":false}],"qosClass":"BestEffort"}},"MirrorPod":null,"RunningPod":null,"KillPodOptions":null}}
同步状态到api-server
configCh的消息 {"Pods":[{"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lls":{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:34Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:34Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.31","podIPs":[{"ip":"10.10.1.31"}],"startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-29T13:38:33Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","started":true}],"qosClass":"BestEffort"}}],"Op":5,"Source":"api"}
podManager更新Pod缓存 {"metadata":{"name":"busybox2","namespace":"default","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","resourceVersion":"88575","creationTimestamp":"2022-06-29T13:38:17Z","lab:{"app":"busybox"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2022-06-29T13:38:17Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"busybox\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-mv4cg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"busybox","image":"busybox","command":["/bin/sh","-c","sleep 36000;"],"resources":{},"volumeMounts":[{"name":"kube-api-access-mv4cg","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"node1","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:34Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:34Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2022-06-29T13:38:17Z"}],"hostIP":"192.168.100.101","podIP":"10.10.1.31","podIPs":[{"ip":"10.10.1.31"}],"startTime":"2022-06-29T13:38:17Z","containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-29T13:38:33Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","started":true}],"qosClass":"BestEffort"}}
2022-06-29 21:38:35.37883752 +0800 CST m=+116.949408051  执行syncPod，当前container状态： {"ID":"95bdac1a-7a17-40c9-b75d-e07816859760","Name":"busybox2","Namespace":"default","IPs":["10.10.1.31ainerStatuses":[{"ID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","Name":"busybox","State":"running","CreatedAt":"2022-06-29T21:38:33.566069909+08:00","StartedAt":"2022-06-29T21:38:33.644378832+08:00","FinishedAt":"0001-01-01T00:00:00Z","ExitCode":0,"Image":"busybox:latest","ImageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","Hash":534858999,"RestartCount":0,"Reason":"","Message":""}],"SandboxStatuses":[{"id":"3e758a520626620d85d010898ee1e1e4735fa1778d83886289e3ec776cfef1e5","metadata":{"name":"busybox2","uid":"95bdac1a-7a17-40c9-b75d-e07816859760","namespace":"default"},"created_at":1656509897627665587,"network":{"ip":"10.10.1.31"},"linux":{"namespaces":{"options":{"pid":1}}},"labels":{"app":"busybox","io.kubernetes.pod.name":"busybox2","io.kubernetes.pod.namespace":"default","io.kubernetes.pod.uid":"95bdac1a-7a17-40c9-b75d-e07816859760"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n","kubernetes.io/config.seen":"2022-06-29T21:38:17.313275109+08:00","kubernetes.io/config.source":"api"}}]}
2022-06-29 21:38:35.379173583 +0800 CST m=+116.949744112 设置Pod状态： {"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"t"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":null},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":null}],"hostIP":"192.168.100.101","podIP":"10.10.1.31","podIPs":[{"ip":"10.10.1.31"}],"containerStatuses":[{"name":"busybox","state":{"running":{"startedAt":"2022-06-29T13:38:33Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"busybox:latest","imageID":"docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678","containerID":"docker://c597d478635edbb03d4d559b42ef5fdac5d8fd30432986e7672c0f3e72048cb4","started":true}],"qosClass":"BestEffort"}
2022-06-29 21:38:35.379533924 +0800 CST m=+116.950104452 调用CRI SyncPod
2022-06-29 21:38:35.38163804 +0800 CST m=+116.952208573 调用CRI SyncPod完毕
处理完一个PodWork，无任务在排队
# 创建Pod流程

## 流程
1. api-server新建Pod资源，并且调度到该节点
2. 主循环收到configCh消息，消息类型为ADD，其内PodStatus为Pendding，其余状态为空
    ```json
    {
        "Pods": [
            {
                "metadata": {
                    "name": "busybox2",
                    "namespace": "default",
                    "uid": "52baae0f-de0b-43ab-be1f-7a4774305e97",
                    "resourceVersion": "88050",
                    "creationTimestamp": "2022-06-29T02:39:03Z",
                    "lls": {
                        "app": "busybox"
                    },
                    "annotations": {
                        "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                        "kubernetes.io/config.seen": "2022-06-29T10:39:03.409542655+08:00",
                        "kubernetes.io/config.source": "api"
                    },
                },
                "spec": {
                    "volumes": [
                        {
                            "name": "kube-api-access-wnlr8",
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
                                    "name": "kube-api-access-wnlr8",
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
                    "phase": "Pending",
                    "qosClass": "BestEffort"
                }
            }
        ],
        "Op": 1,
        "Source": "api"
    }
    ```
3. 更新PodManager缓存，交由PodWorkes处理
4. PodWorkes获取从podCache中获取kubecontainer.PodStatus，此时会返回空状态，调用Kubelet.syncPod将其传入
    ```json
    {
        "ID": "52baae0f-de0b-43ab-be1f-7a4774305e97",
        "Name": "",
        "Namespace": "",
        "IPs": null,
        "ContainerStatuses": null,
        "SandboxStatuses": null
    }
    ```
5. Kubelet.syncPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，此时containerStatuses中state为waiting。更新StatusManager，由于没有旧状态会进行同步。产生一个异步流程
    ```json
    {
        "phase": "Pending",
        "conditions": [
            {
                "type": "Initialized",
                "status": "True",
                "lastProbeTime": null,
                "lastTransitionTime": null
            },
            {
                "type": "Ready",
                "status": "False",
                "lastProbeTime": null,
                "lastTtionTime": null,
                "reason": "ContainersNotReady",
                "message": "containers with unready status: [busybox]"
            },
            {
                "type": "ContainersReady",
                "status": "False",
                "lastProbeTime": null,
                "lastTransitionTime": null,
                "reason": "ContainersNotReady",
                "message": "containers with unready status: [busybox]"
            },
            {
                "type": "PodScheduled",
                "status": "True",
                "lastProbeTime": null,
                "lastTransitionTime": null
            }
        ],
        "hostIP": "192.168.100.101",
        "containerStatuses": [
            {
                "name": "busybox",
                "state": {
                    "waiting": {
                        "reason": "ContainerCreating"
                    }
                },
                "lastState": {},
                "ready": false,
                "restartCount": 0,
                "image": "busybox",
                "imageID": "",
                "started": false
            }
        ],
        "qosClass": "BestEffort"
    }
    ```
   1. StatusManager将状态同步至api-server
   2. 主循环收到configCh消息，消息类型为RECONCILE，状态为上一步同步至api-server的状态
        ```json
        {
            "Pods": [
                {
                    "metadata": {
                        "name": "busybox2",
                        "namespace": "default",
                        "uid": "52baae0f-de0b-43ab-be1f-7a4774305e97",
                        "resourceVersion": "88050",
                        "creationTimestamp": "2022-06-29T02:39:03Z",
                        "lls": {
                            "app": "busybox"
                        },
                        "annotations": {
                            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                            "kubernetes.io/config.seen": "2022-06-29T10:39:03.409542655+08:00",
                            "kubernetes.io/config.source": "api"
                        },
                    },
                    "spec": {
                        "volumes": [
                            {
                                "name": "kube-api-access-wnlr8",
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
                                        "name": "kube-api-access-wnlr8",
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
                        "phase": "Pending",
                        "conditions": [
                            {
                                "type": "Initialized",
                                "status": "True",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:03Z"
                            },
                            {
                                "type": "Ready",
                                "status": "False",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:03Z",
                                "reason": "ContainersNotReady",
                                "message": "containers with unready status: [busybox]"
                            },
                            {
                                "type": "ContainersReady",
                                "status": "False",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:03Z",
                                "reason": "ContainersNotReady",
                                "message": "containers with unready status: [busybox]"
                            },
                            {
                                "type": "PodScheduled",
                                "status": "True",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:03Z"
                            }
                        ],
                        "hostIP": "192.168.100.101",
                        "startTime": "2022-06-29T02:39:03Z",
                        "containerStatuses": [
                            {
                                "name": "busybox",
                                "state": {
                                    "waiting": {
                                        "reason": "ContainerCreating"
                                    }
                                },
                                "lastState": {},
                                "ready": false,
                                "restartCount": 0,
                                "image": "busybox",
                                "imageID": "",
                                "started": false
                            }
                        ],
                        "qosClass": "BestEffort"
                    }
                }
            ],
            "Op": 5,
            "Source": "api"
        }
        ```
   3. 更新PodManager缓存，然后操作被PodWorkers暂存
   4. PodWorkers上次Work执行完毕后，获取从podCache中获取kubecontainer.PodStatus，此时如果容器还未启动会返回空状态，调用Kubelet.syncPod将其传入。如果容器已启动，进入第10步。
   5. Kubelet.syncPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，此时containerStatuses中state为waiting。更新StatusManager，由于状态与旧状态一致，不进行同步。
6. Kubelet.syncPod中调用CRI计算要创建的容器，containerRuntime开始创建容器(阻塞操作，需要较长时间)
7. 容器创建好后，PLEG收到消息，更新PodCache，发送消息到主循环(被PodWorkers暂存)
8. 主循环收到PLEG消息，类型为ContainerStarted
    ```json
    {"ID":"52baae0f-de0b-43ab-be1f-7a4774305e97","Type":"ContainerStarted","Data":"42066240c4b17a4d8f79a5a3eb0365b357ddddf6bbaff2fd2b7962f7186a4f85"}
    ```
9.  交由PodWorkes处理
10. PodWorkes获取从podCache中获取kubecontainer.PodStatus，调用Kubelet.syncPod将其传入
    ```json
    {
        "ID": "52baae0f-de0b-43ab-be1f-7a4774305e97",
        "Name": "busybox2",
        "Namespace": "default",
        "IPs": [
            "10.10.1.24"
        ],
        "ContainerStatuses": [
            {
                "ID": "docker://0c657776f09b523fae8ab482b5ce4ddc34bac88335b759ceb87eabc3",
                "Name": "busybox",
                "State": "running",
                "CreatedAt": "2022-06-29T10:39:05.063168708+08:00",
                "StartedAt": "2022-06-29T10:39:05.138930533+08:00",
                "FinishedAt": "0001-01-01T00:00:00Z",
                "ExitCode": 0,
                "Image": "busybox:latest",
                "ImageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
                "Hash": 912014391,
                "RestartCount": 0,
                "Reason": "",
                "Message": ""
            }
        ],
        "SandboxStatuses": [
            {
                "id": "42066240c4b17a4d8f79a5a3eb0365b357ddddf6bbaff2fd2b7962f7186a4f85",
                "metadata": {
                    "name": "busybox2",
                    "uid": "52baae0f-de0b-43ab-be1f-7a4774305e97",
                    "namespace": "default"
                },
                "created_at": 1656470343719492646,
                "network": {
                    "ip": "10.10.1.24"
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
                    "io.kubernetes.pod.uid": "52baae0f-de0b-43ab-be1f-7a4774305e97"
                },
                "annotations": {
                    "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                    "kubernetes.io/config.seen": "2022-06-29T10:39:03.409542655+08:00",
                    "kubernetes.io/config.source": "api"
                }
            }
        ]
    }
    ```
11. Kubelet.syncPod中使用kubecontainer.PodStatus计算出corev1.PodStatus，此时containerStatuses中state为running。更新StatusManager，由于状态与旧状态不一致，进行同步。产生异步流程
    ```json
    {
        "phase": "Running",
        "conditions": [
            {
                "type": "Initialized",
                "status": "True",
                "lastProbeTime": null,
                "lastTransitionTime": null
            },
            {
                "type": "Ready",
                "status": "True",
                "lastProbeTime": null,
                "lastTrionTime": null
            },
            {
                "type": "ContainersReady",
                "status": "True",
                "lastProbeTime": null,
                "lastTransitionTime": null
            },
            {
                "type": "PodScheduled",
                "status": "True",
                "lastProbeTime": null,
                "lastTransitionTime": null
            }
        ],
        "hostIP": "192.168.100.101",
        "podIP": "10.10.1.24",
        "podIPs": [
            {
                "ip": "10.10.1.24"
            }
        ],
        "containerStatuses": [
            {
                "name": "busybox",
                "state": {
                    "running": {
                        "startedAt": "2022-06-29T02:39:05Z"
                    }
                },
                "lastState": {},
                "ready": true,
                "restartCount": 0,
                "image": "busybox:latest",
                "imageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
                "containerID": "docker://0c657775dea88b36f09b523fae8ab482b5ce4ddc34bac88335b759ceb87eabc3",
                "started": true
            }
        ],
        "qosClass": "BestEffort"
    }
    ```
    1. StatusManager将状态同步至api-server。
    2. 主循环收到configCh消息，消息类型为RECONCILE，状态为上一步同步至api-server的状态
        ```json
        {
            "Pods": [
                {
                    "metadata": {
                        "name": "busybox2",
                        "namespace": "default",
                        "uid": "52baae0f-de0b-43ab-be1f-7a4774305e97",
                        "resourceVersion": "88050",
                        "creationTimestamp": "2022-06-29T02:39:03Z",
                        "lls": {
                            "app": "busybox"
                        },
                        "annotations": {
                            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"busybox\"},\"name\":\"busybox2\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"/bin/sh\",\"-c\",\"sleep 36000;\"],\"image\":\"busybox\",\"name\":\"busybox\"}],\"nodeName\":\"node1\"}}\n",
                            "kubernetes.io/config.seen": "2022-06-29T10:39:03.409542655+08:00",
                            "kubernetes.io/config.source": "api"
                        },
                    },
                    "spec": {
                        "volumes": [
                            {
                                "name": "kube-api-access-wnlr8",
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
                                        "name": "kube-api-access-wnlr8",
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
                                "lastTransitionTime": "2022-06-29T02:39:03Z"
                            },
                            {
                                "type": "Ready",
                                "status": "True",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:05Z"
                            },
                            {
                                "type": "ContainersReady",
                                "status": "True",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:05Z"
                            },
                            {
                                "type": "PodScheduled",
                                "status": "True",
                                "lastProbeTime": null,
                                "lastTransitionTime": "2022-06-29T02:39:03Z"
                            }
                        ],
                        "hostIP": "192.168.100.101",
                        "podIP": "10.10.1.24",
                        "podIPs": [
                            {
                                "ip": "10.10.1.24"
                            }
                        ],
                        "startTime": "2022-06-29T02:39:03Z",
                        "containerStatuses": [
                            {
                                "name": "busybox",
                                "state": {
                                    "running": {
                                        "startedAt": "2022-06-29T02:39:05Z"
                                    }
                                },
                                "lastState": {},
                                "ready": true,
                                "restartCount": 0,
                                "image": "busybox:latest",
                                "imageID": "docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678",
                                "containerID": "docker://0c657775dea88b36f09b523fae8ab482b5ce4ddc34bac88335b759ceb87eabc3",
                                "started": true
                            }
                        ],
                        "qosClass": "BestEffort"
                    }
                }
            ],
            "Op": 5,
            "Source": "api"
        }
        ```
    3. 更新PodManager缓存
    4. PodWorkes获取从podCache中获取kubecontainer.PodStatus，此时containerStatuses中state为running。更新StatusManager，由于状态与旧状态一致，不进行同步。
    5. 容器启动完成，并与api-server同步状态完毕
12. Kubelet.syncPod中调用CRI计算要创建的容器，计算结果无需要创建的容器