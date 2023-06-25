# EventCorrelate
EventCorrelate使用LRU(最近最少使用)算法来淘汰事件缓存

## 结构体
```go
// vendor/k8s.io/client-go/tools/record/events_cache.go
type EventCorrelator struct {
	// the function to filter the event
	// 过滤事件的函数
	filterFunc EventFilterFunc
	// the object that performs event aggregation
	// 事件聚合器
	aggregator *EventAggregator
	// the object that observes events as they come through
	// 事件记录器，缓存最近的事件
	logger *eventLogger
}
```

## 方法
```go
// vendor/k8s.io/client-go/tools/record/events_cache.go
func (c *EventCorrelator) EventCorrelate(newEvent *v1.Event) (*EventCorrelateResult, error) {
	if newEvent == nil {
		return nil, fmt.Errorf("event is nil")
	}
	aggregateEvent, ckey := c.aggregator.EventAggregate(newEvent)
	observedEvent, patch, err := c.logger.eventObserve(aggregateEvent, ckey)
	if c.filterFunc(observedEvent) {
		return &EventCorrelateResult{Skip: true}, nil
	}
	return &EventCorrelateResult{Event: observedEvent, Patch: patch}, err
}

// UpdateState based on the latest observed state from server
func (c *EventCorrelator) UpdateState(event *v1.Event) {
	c.logger.updateState(event)
}
```