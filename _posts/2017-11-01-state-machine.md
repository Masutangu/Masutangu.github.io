---
layout: post
date: 2017-11-01T08:33:41+08:00
title: 游戏开发中的状态机
category: 工作
---

这阵子工作的内容有用到状态机，感觉挺有意思。正好好久没写博客了，今天也来写一篇总结下。

# 前言

用状态机来实现业务模型，有以下几点好处：

* 不需要写一大坨 if-else 或 switch case。代码逻辑结构清晰，也更便于调试
* 代码阅读起来更加友好，方便其他读者理解整个业务逻辑

状态机可以划分为下面三个模块：

* **状态集**：总共包括哪些状态
* **事件（条件）**：事件会触发状态机的状态发生变化
* **动作**：事件发生后执行的动作，可以变迁到新状态，也可以维持当前状态

# 实现

## 一个简单的状态机的实现

```golang
// example 1. simple state machine
// 事件 interface
type Event interface {
	EventId() int64
}

type State int

// 事件处理函数 返回触发事件后的下一个状态 
type Handler func(prevState State, event Event) State

type StateMachine struct {
	currState    State
	handlers map[int64]Handler
}

// 添加事件处理方法 
func (s *StateMachine) AddHandler(eventId int64, handler Handler) {
	s.handlers[eventId] = handler
}

// 事件处理
func (s *StateMachine) Call(event Event)  {
	fmt.Println("original state: ", s.currState)
	if handler, ok := s.handlers[event.EventId()]; ok {
      s.currState = handler(s.currState, event)  // 调用对应的 handler 更新状态机的状态
	}
	fmt.Println("new state: ", s.currState)
}
```

```Handler``` 为事件处理函数，输入参数为当前状态和事件，返回处理后的新状态。状态机根据事件找到对应的```Handler```，```Handler``` 根据当前状态和触发的事件，返回下一个新状态，由状态机更新。

下面看看一个开关的例子，初始状态为关，按下按钮状态由关变为开，再次按下由开变为关。

```golang
// example 1. simple state machine
const (
  EVENT_PRESS = iota
)

var (
	Off = State(0)  // 定义关闭状态
	On  = State(1)  // 定义开启状态 
)

// 定义 press 事件
type PressEvent struct {
}

func (event *PressEvent) EventId() int64 {
	return EVENT_PRESS
}

// 定义事件 Handler
func PressButton(prevState State, event Event) State {
  if prevState == Off {
    return On
  } else {
    return Off
  }
}

func main() {
  stateMachine := StateMachine{
      currState:    Off,  // 初始状态为关闭
      handlers: make(map[int64]Handler),
  }

  stateMachine.AddHandler(EVENT_PRESS, PressButton)
  stateMachine.Call(&PressEvent{}) // 按下后变成开
  stateMachine.Call(&PressEvent{}) // 再次按下变为关闭
}
```

程序输出：

```
original state:  0
new state:  1
original state:  1
new state:  0
```

这个实现很简单，但不足之处在于状态变迁逻辑都放在```Handler```去实现了。对于读代码的人来说，需要读每个Handler的代码，才能整理出整个状态变迁图。

## 一个稍微复杂点的状态机

我们希望可以把状态变迁以更直观的方式表现出来，让读者一看就知道状态是如何流转的。

```golang 
// example 2. a little more complicate state machine

type Event interface {
	EventId() int64
}

type State int

// 事件处理函数 返回 true 表示可以变迁到下一个状态 返回 false 表示维持当前状态
type Handler func(prevState State, event Event) bool

type StateMachine struct {
	currState    State
	handlers map[int64]Handler
  transitions map[State]map[int64]State
}

// 添加事件处理方法 
func (s *StateMachine) AddHandler(eventId int64, handler Handler) {
	s.handlers[eventId] = handler
}

// 添加状态变迁
func (s *StateMachine) AddTransition(originState State, eventId int64, destState State) {
  if trans, ok := s.transitions[originState]; !ok {
     s.transitions[originState] = map[int64]State{eventId: destState}
  } else {
     trans[eventId] = destState
  }
}

// 事件处理
func (s *StateMachine) Call(event Event) {
  fmt.Println("original state: ", s.currState)  
  if handler, ok := s.handlers[event.EventId()]; ok { // 首先找到事件的handler
    if handler(s.currState, event) { // 如果事件Handler返回true 则执行状态变迁
      if trans, ok := s.transitions[s.currState]; ok {
        if newState, ok := trans[event.EventId()]; ok { 
          s.currState = newState  // 执行状态变迁
        }
      }
    }
  }
  fmt.Println("new state: ", s.currState) 
}
```

这个状态机用```transitions``` 结构来记录状态流转的关系。初始化时调用方调用```AddTransition```方法来添加状态变迁。请看下面的例子：

```golang
// example 2. a little more complicate state machine

const (
  EVENT_PRESS = iota
)

var (
  Off = State(0)  // 关闭
  On  = State(1)  // 开启 
  COUNT = 0
)


type PressEvent struct {
}

func (event *PressEvent) EventId() int64 {
	return EVENT_PRESS
}

// 定义事件 Handler
func PressButton(prevState State, event Event) bool {
  COUNT += 1
  if COUNT % 2 == 0 { // 按两下才切换到新状态
    return true
  }
  return false // 只按一次维持当前状态
}

func main() {
  stateMachine := StateMachine{
      currState:    Off,  // 初始状态为关闭
      handlers: make(map[int64]Handler),
      transitions: make(map[State]map[int64]State),
  }

  stateMachine.AddHandler(EVENT_PRESS, PressButton)
  stateMachine.AddTransition(Off, EVENT_PRESS, On)
  stateMachine.AddTransition(On, EVENT_PRESS, Off)
  stateMachine.Call(&PressEvent{}) // 按一次状态不变
  stateMachine.Call(&PressEvent{}) // 按两次变成 off 状态
}
```

通过```stateMachine.AddTransition(Off, EVENT_PRESS, On)```就可以清晰的知道 Press 事件可能会让状态 Off 切换到 状态 On，尽管 Press 事件发生后还是有可能维持当前状态不变（当 Handler 返回 false 时）。


程序输出：

```
original state:  0
new state:  0
original state:  0
new state:  1
```

## 更模块化的状态机

最后我们来实现一个更模块化的状态机，把事件和条件这两个逻辑彻底分开。当你在事件触发时，还需要做逻辑判断才能确定是否发生状态变迁时，建议将事件处理和条件判断剥离开来。

```golang
// example 3. more modular state machine

type Event interface {
	EventId() int64
}

type State int

// 事件处理 Handler
type Handler interface {
	Process(prevState State, event Event)  // 只处理事件
	Check() bool                           // 处理完判断下是否应该做状态切换
}

type StateMachine struct {
	currState    State
	handlers 			map[int64]Handler
  transitions 	map[State]map[int64]State
}

// 添加事件处理方法 
func (s *StateMachine) AddHandler(eventId int64, handler Handler) {
	s.handlers[eventId] = handler
}

// 添加状态变迁
func (s *StateMachine) AddTransition(originState State, eventId int64, destState State) {
  if trans, ok := s.transitions[originState]; !ok {
     s.transitions[originState] = map[int64]State{eventId: destState}
  } else {
     trans[eventId] = destState
  }
}

// 事件处理
func (s *StateMachine) Call(event Event) {
  fmt.Println("original state: ", s.currState)  
  if handler, ok := s.handlers[event.EventId()]; ok { // 首先找到事件的handler
    handler.Process(s.currState, event) 
    
    if handler.Check() {
      if trans, ok := s.transitions[s.currState]; ok {
        if newState, ok := trans[event.EventId()]; ok { 
          s.currState = newState  // 执行状态变迁
        }
      }
    }
  }
  fmt.Println("new state: ", s.currState) 
}
```

定义```Handler```结构体，有两个接口：```Process```只负责事件引发的逻辑处理， ```Check```判断逻辑处理后是否应该做状态变迁。

```go
// example 3. more modular state machine

const (
  EVENT_PRESS = iota
)

var (
  Off = State(0)  // 关闭
  On  = State(1)  // 开启 
  COUNT = 0
)

type PressEvent struct {
}

func (event *PressEvent) EventId() int64 {
	return EVENT_PRESS
}

type PressEventHandler struct {

}
// Process 只处理事件带来的内部变量的变化
func (h *PressEventHandler) Process(prevState State, event Event) {
  COUNT += 1
}


// Check 判断是否应该做状态切换
func (h *PressEventHandler) Check() bool {
  if COUNT % 2 == 0 { // 按两下才有作用
    return true
  }
  return false
}

func main() {
  stateMachine := StateMachine{
      currState:    Off,  // 初始状态为关闭
      handlers: make(map[int64]Handler),
      transitions: make(map[State]map[int64]State),
  }

  stateMachine.AddHandler(EVENT_PRESS, &PressEventHandler{})
  stateMachine.AddTransition(Off, EVENT_PRESS, On)
  stateMachine.AddTransition(On, EVENT_PRESS, Off)
  stateMachine.Call(&PressEvent{}) // 按一次状态不变
  stateMachine.Call(&PressEvent{}) // 按两次变成 off 状态
}
```

程序输出：
```
original state:  0
new state:  0
original state:  0
new state:  1
```

## 一些补充

可以把 ```State``` 定义成 interface，提供 ```Enter``` 和 ```Leave``` 接口：

```golang
type State interface {
  Enter() 
  Leave()
}

// 事件处理
func (s *StateMachine) Call(event Event) {
  fmt.Println("original state: ", s.currState)  
  if handler, ok := s.handlers[event.EventId()]; ok { // 首先找到事件的handler
    handler.Process(s.currState, event) 

    if handler.Check() {
      if trans, ok := s.transitions[s.currState]; ok {
        if newState, ok := trans[event.EventId()]; ok { 
          s.currState.Leave()     // 离开当前状态 调用 Leave 
          s.currState = newState  // 执行状态变迁
          s.currState.Enter()     // 进入新状态 调用 Enter
        }
      }
    }
  }
  fmt.Println("new state: ", s.currState) 
}
```

## 与 goroutine 的结合

涉及到多个 goroutine 时，总会面临数据竞争的问题。通过 channel 来传递事件，由状态机处理，可以让代码变得清晰，避免加锁。

```golang

type Room struct {
  eventCh chan Event // 通过channel 传递事件给状态机
  st *StateMachine
}

func (room *Room) Process() {
  for e := range room.eventCh {
    room.st.Call(e)
  }
}

func (room *Room) DispatchEvent(event Event) {
  room.eventCh <- event
}

func main() {
  stateMachine := StateMachine{
      currState:    Off,  // 初始状态为关闭
      handlers: make(map[int64]Handler),
      transitions: make(map[State]map[int64]State),
  }

  stateMachine.AddHandler(EVENT_PRESS, &PressEventHandler{})
  stateMachine.AddTransition(Off, EVENT_PRESS, On)
  stateMachine.AddTransition(On, EVENT_PRESS, Off)
	
  room := Room{st: &stateMachine, eventCh: make(chan Event)}

  go room.DispatchEvent(&PressEvent{})
  go room.DispatchEvent(&PressEvent{})
  go room.DispatchEvent(&PressEvent{})

  room.Process()  // just sample code
}
```

即使是多个 goroutine 并发抛出事件，状态机只从```eventCh```中串行的取出事件并处理，处理过程中不需要对数据加锁。 