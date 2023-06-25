# Options默认值

```json
{
    "Generic": {
        "Port": 0,
        "Address": "0.0.0.0",
        "MinResyncPeriod": "12h0m0s",
        "ClientConnection": {
            "Kubeconfig": "",
            "AcceptContentTypes": "",
            "ContentType": "application/vnd.kubernetes.protobuf",
            "QPS": 20,
            "Burst": 30
        },
        "ControllerStartInterval": "0s",
        "LeaderElection": {
            "LeaderElect": true,
            "LeaseDuration": "15s",
            "RenewDeadline": "10s",
            "RetryPeriod": "2s",
            "ResourceLock": "leases",
            "ResourceName": "kube-controller-manager",
            "ResourceNamespace": "kube-system"
        },
        "Controllers": [
            "*"
        ],
        "LeaderMigrationEnabled": false,
        "Debugging": {
            "EnableProfiling": true,
            "EnableContentionProfiling": false
        },
        "LeaderMigration": {
            "Enabled": false,
            "ControllerMigrationConfig": ""
        }
    },
    "KubeCloudShared": {
        "ExternalCloudVolumePlugin": "",
        "UseServiceAccountCredentials": false,
        "AllowUntaggedCloud": false,
        "RouteReconciliationPeriod": "10s",
        "NodeMonitorPeriod": "5s",
        "ClusterName": "kubernetes",
        "ClusterCIDR": "",
        "AllocateNodeCIDRs": false,
        "CIDRAllocatorType": "",
        "ConfigureCloudRoutes": true,
        "NodeSyncPeriod": "0s",
        "CloudProvider": {
            "Name": "",
            "CloudConfigFile": ""
        }
    },
    "ServiceController": {
        "ConcurrentServiceSyncs": 1
    },
    "AttachDetachController": {
        "DisableAttachDetachReconcilerSync": false,
        "ReconcilerSyncLoopPeriod": "1m0s"
    },
    "CSRSigningController": {
        "ClusterSigningCertFile": "",
        "ClusterSigningKeyFile": "",
        "KubeletServingSignerConfiguration": {
            "CertFile": "",
            "KeyFile": ""
        },
        "KubeletClientSignerConfiguration": {
            "CertFile": "",
            "KeyFile": ""
        },
        "KubeAPIServerClientSignerConfiguration": {
            "CertFile": "",
            "KeyFile": ""
        },
        "LegacyUnknownSignerConfiguration": {
            "CertFile": "",
            "KeyFile": ""
        },
        "ClusterSigningDuration": "8760h0m0s"
    },
    "DaemonSetController": {
        "ConcurrentDaemonSetSyncs": 2
    },
    "DeploymentController": {
        "ConcurrentDeploymentSyncs": 5,
        "DeploymentControllerSyncPeriod": "30s"
    },
    "StatefulSetController": {
        "ConcurrentStatefulSetSyncs": 5
    },
    "DeprecatedFlags": {
        "DeletingPodsQPS": 0,
        "DeletingPodsBurst": 0,
        "RegisterRetryCount": 10
    },
    "EndpointController": {
        "ConcurrentEndpointSyncs": 5,
        "EndpointUpdatesBatchPeriod": "0s"
    },
    "EndpointSliceController": {
        "ConcurrentServiceEndpointSyncs": 5,
        "MaxEndpointsPerSlice": 100,
        "EndpointUpdatesBatchPeriod": "0s"
    },
    "EndpointSliceMirroringController": {
        "MirroringConcurrentServiceEndpointSyncs": 5,
        "MirroringMaxEndpointsPerSubset": 1000,
        "MirroringEndpointUpdatesBatchPeriod": "0s"
    },
    "EphemeralVolumeController": {
        "ConcurrentEphemeralVolumeSyncs": 5
    },
    "GarbageCollectorController": {
        "EnableGarbageCollector": true,
        "ConcurrentGCSyncs": 20,
        "GCIgnoredResources": [
            {
                "Group": "",
                "Resource": "events"
            }
        ]
    },
    "HPAController": {
        "HorizontalPodAutoscalerSyncPeriod": "15s",
        "HorizontalPodAutoscalerUpscaleForbiddenWindow": "3m0s",
        "HorizontalPodAutoscalerDownscaleForbiddenWindow": "5m0s",
        "HorizontalPodAutoscalerDownscaleStabilizationWindow": "5m0s",
        "HorizontalPodAutoscalerTolerance": 0.1,
        "HorizontalPodAutoscalerCPUInitializationPeriod": "5m0s",
        "HorizontalPodAutoscalerInitialReadinessDelay": "30s"
    },
    "JobController": {
        "ConcurrentJobSyncs": 5
    },
    "CronJobController": {
        "ConcurrentCronJobSyncs": 5
    },
    "NamespaceController": {
        "NamespaceSyncPeriod": "5m0s",
        "ConcurrentNamespaceSyncs": 10
    },
    "NodeIPAMController": {
        "ServiceCIDR": "",
        "SecondaryServiceCIDR": "",
        "NodeCIDRMaskSize": 0,
        "NodeCIDRMaskSizeIPv4": 0,
        "NodeCIDRMaskSizeIPv6": 0
    },
    "NodeLifecycleController": {
        "EnableTaintManager": true,
        "NodeEvictionRate": 0,
        "SecondaryNodeEvictionRate": 0,
        "NodeStartupGracePeriod": "1m0s",
        "NodeMonitorGracePeriod": "40s",
        "PodEvictionTimeout": "5m0s",
        "LargeClusterSizeThreshold": 0,
        "UnhealthyZoneThreshold": 0
    },
    "PersistentVolumeBinderController": {
        "PVClaimBinderSyncPeriod": "15s",
        "VolumeConfiguration": {
            "EnableHostPathProvisioning": false,
            "EnableDynamicProvisioning": true,
            "PersistentVolumeRecyclerConfiguration": {
                "MaximumRetry": 3,
                "MinimumTimeoutNFS": 300,
                "PodTemplateFilePathNFS": "",
                "IncrementTimeoutNFS": 30,
                "PodTemplateFilePathHostPath": "",
                "MinimumTimeoutHostPath": 60,
                "IncrementTimeoutHostPath": 30
            },
            "FlexVolumePluginDir": "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/"
        },
        "VolumeHostCIDRDenylist": null,
        "VolumeHostAllowLocalLoopback": true
    },
    "PodGCController": {
        "TerminatedPodGCThreshold": 12500
    },
    "ReplicaSetController": {
        "ConcurrentRSSyncs": 5
    },
    "ReplicationController": {
        "ConcurrentRCSyncs": 5
    },
    "ResourceQuotaController": {
        "ResourceQuotaSyncPeriod": "5m0s",
        "ConcurrentResourceQuotaSyncs": 5
    },
    "SAController": {
        "ServiceAccountKeyFile": "",
        "ConcurrentSATokenSyncs": 5,
        "RootCAFile": ""
    },
    "TTLAfterFinishedController": {
        "ConcurrentTTLSyncs": 5
    },
    "SecureServing": {
        "BindAddress": "0.0.0.0",
        "BindPort": 10257,
        "BindNetwork": "",
        "Required": false,
        "ExternalAddress": "",
        "Listener": null,
        "ServerCert": {
            "CertKey": {
                "CertFile": "",
                "KeyFile": ""
            },
            "CertDirectory": "",
            "PairName": "kube-controller-manager",
            "GeneratedCert": null,
            "FixtureDirectory": ""
        },
        "SNICertKeys": null,
        "CipherSuites": null,
        "MinTLSVersion": "",
        "HTTP2MaxStreamsPerConnection": 0,
        "PermitPortSharing": false,
        "PermitAddressSharing": false
    },
    "Authentication": {
        "RemoteKubeConfigFile": "",
        "RemoteKubeConfigFileOptional": true,
        "CacheTTL": 10000000000,
        "ClientCert": {
            "ClientCA": "",
            "CAContentProvider": null
        },
        "RequestHeader": {
            "ClientCAFile": "",
            "UsernameHeaders": [
                "x-remote-user"
            ],
            "GroupHeaders": [
                "x-remote-group"
            ],
            "ExtraHeaderPrefixes": [
                "x-remote-extra-"
            ],
            "AllowedNames": null
        },
        "SkipInClusterLookup": false,
        "TolerateInClusterLookupFailure": false,
        "WebhookRetryBackoff": {
            "Duration": 500000000,
            "Factor": 1.5,
            "Jitter": 0.2,
            "Steps": 5,
            "Cap": 0
        },
        "TokenRequestTimeout": 10000000000
    },
    "Authorization": {
        "RemoteKubeConfigFile": "",
        "RemoteKubeConfigFileOptional": true,
        "AllowCacheTTL": 10000000000,
        "DenyCacheTTL": 10000000000,
        "AlwaysAllowPaths": [
            "/healthz",
            "/readyz",
            "/livez"
        ],
        "AlwaysAllowGroups": [
            "system:masters"
        ],
        "ClientTimeout": 10000000000,
        "WebhookRetryBackoff": {
            "Duration": 500000000,
            "Factor": 1.5,
            "Jitter": 0.2,
            "Steps": 5,
            "Cap": 0
        }
    },
    "Metrics": {
        "ShowHiddenMetricsForVersion": "",
        "DisabledMetrics": null,
        "AllowListMapping": null
    },
    "Logs": {
        "Config": {
            "Format": "text",
            "FlushFrequency": 5000000000,
            "Verbosity": 0,
            "VModule": null,
            "Sanitization": false,
            "Options": {
                "JSON": {
                    "SplitStream": false,
                    "InfoBufferSize": "0"
                }
            }
        }
    },
    "Master": "",
    "Kubeconfig": "",
    "ShowHiddenMetricsForVersion": ""
}
```