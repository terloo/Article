# names
names保存了所有插件的名字常量，方便统一使用

```go
// pkg/scheduler/framework/plugins/names/names.go
const (
	PrioritySort                    = "PrioritySort"
	DefaultBinder                   = "DefaultBinder"
	DefaultPreemption               = "DefaultPreemption"
	ImageLocality                   = "ImageLocality"
	InterPodAffinity                = "InterPodAffinity"
	NodeAffinity                    = "NodeAffinity"
	NodeName                        = "NodeName"
	NodePorts                       = "NodePorts"
	NodeResourcesBalancedAllocation = "NodeResourcesBalancedAllocation"
	NodeResourcesFit                = "NodeResourcesFit"
	NodeUnschedulable               = "NodeUnschedulable"
	NodeVolumeLimits                = "NodeVolumeLimits"
	AzureDiskLimits                 = "AzureDiskLimits"
	CinderLimits                    = "CinderLimits"
	EBSLimits                       = "EBSLimits"
	GCEPDLimits                     = "GCEPDLimits"
	PodTopologySpread               = "PodTopologySpread"
	SelectorSpread                  = "SelectorSpread"
	ServiceAffinity                 = "ServiceAffinity"
	TaintToleration                 = "TaintToleration"
	VolumeBinding                   = "VolumeBinding"
	VolumeRestrictions              = "VolumeRestrictions"
	VolumeZone                      = "VolumeZone"
)
```