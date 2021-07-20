# Cycle State

CycleState用来在plugins之间传递数据。所以的Plugin可以往CycleState中保存数据、修改其中的数据或删除数据。可以将CycleState看作是一次调度过程的context。

## CycleState

### 数据结构

往CycleState中保存的对象都要实现StateData接口。

```go
// StateData is a generic type for arbitrary data stored in CycleState.
type StateData interface {
	// Clone is an interface to make a copy of StateData. For performance reasons,
	// clone should make shallow copies for members (e.g., slices or maps) that are not
	// impacted by PreFilter's optional AddPod/RemovePod methods.
	Clone() StateData
}

// StateKey is the type of keys stored in CycleState.
type StateKey string

// CycleState provides a mechanism for plugins to store and retrieve arbitrary data.
// StateData stored by one plugin can be read, altered, or deleted by another plugin.
// CycleState does not provide any data protection, as all plugins are assumed to be
// trusted.
type CycleState struct {
	mx      sync.RWMutex   // 读写锁
	storage map[StateKey]StateData
	// if recordPluginMetrics is true, PluginExecutionDuration will be recorded for this cycle.
	recordPluginMetrics bool
}
```

### 构造函数

```go
// NewCycleState initializes a new CycleState and returns its pointer.
func NewCycleState() *CycleState {
	return &CycleState{
		storage: make(map[StateKey]StateData),
	}
}
```

### 方法

#### ShouldRecordPluginMetrics

`func (c *CycleState) ShouldRecordPluginMetrics() bool`返回是否需要记录PluginExecutionDuration metrics

#### SetRecordPluginMetrics(flag bool)

`func (c *CycleState) SetRecordPluginMetrics(flag bool)`设置recordPluginMetrics

#### Clone

`func (c *CycleState) Clone() *CycleState`克隆一份CycleState，返回其指针。

#### Read

`func (c *CycleState) Read(key StateKey) (StateData, error)`从CycleState中读取键key所对应的Cycle Data，如果键不存在则返回错误。

#### Write

`func (c *CycleState) Write(key StateKey, val StateData)`将键值对(key, val)保存到CycleState中

#### Delete

`func (c *CycleState) Delete(key StateKey)`将键key对应的值保存到CycleState中。

