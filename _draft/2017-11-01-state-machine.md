---
layout: post
date: 2017-11-01T08:33:41+08:00
title: 游戏开发中的状态机
category: 工作
---

这阵子工作的内容涉及到状态机，这里做下总结。

# 前言

用状态机来实现业务模型，有以下几点好处：

* 不需要在写一大坨 if-else 或 switch case
* 逻辑清晰，尤其是在新增/改动状态的时候，也更便于调试
* 代码友好，对整个业务逻辑理解更为整体

状态机可以划分为下面三个模块：

* 状态集：总共包括哪些状态
* 事件（条件）：通过事件触发状态机的状态发生变化
* 动作：条件满足后执行的动作，可以变迁到新状态，也可以维持当前状态

# 实现

## 一个简单的状态机的实现

```go
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
	if handler, ok := s.handlers[event.EventId()]; ok {
      fmt.Println("original state: ", s.currState)
      s.currState = handler(s.currState, event)  // 调用对应的 handler 更新状态机的状态
      fmt.Println("new state: ", s.currState)
	}
}
```

```Handler``` 为事件处理函数，输入参数为当前状态和事件，返回处理后的新状态。状态机根据事件找到对应的```Handler```，```Handler``` 根据当前状态和触发的事件，返回下一个新状态，由状态机更新。

下面看看一个开关的例子，初始状态为关，按下按钮状态由关变为开，再次按下由开变为关。

```go
// example 
const (
  EVENT_PRESS = iota
)

var (
	Off = State(0)  // 关闭
	On  = State(1)  // 开启 
)

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

我们希望可以对整个状态机的变迁一目了然

```go

type Event interface {
	EventId() int64
}

type State int

// 事件处理函数 返回 true 表示可以变迁到下一个状态
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
	s.transitions[originState][eventId] = destState
}

// 事件处理
func (s *StateMachine) Call(event Event) {
  if handler, ok := s.handlers[event.EventId()]; ok { // 首先找到事件的handler
    if handler(s.currState, event) { // 如果事件Handler返回true 则执行状态变迁
      if trans, ok := s.transitions[s.currState]; ok {
        if newState, ok := trans[event.EventId()]; ok { 
          s.currState = newState  // 执行状态变迁
        }
      }
    }
  }
}
```

```go
// example 
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
  if COUNT % 2 == 0 {
    return true
  }
  return false
}

func main() {
	stateMachine := StateMachine{
			currState:    Off,  // 初始状态为关闭
			handlers: make(map[int64]Handler),
      transitions: make(map[State]map[int64]State)
	  }

	stateMachine.AddHandler(EVENT_PRESS, PressButton)
	stateMachine.Call(&PressEvent{}) // 按下后变成开
	stateMachine.Call(&PressEvent{}) // 再次按下变为关闭
}
```

## 更复杂的状态机

把事件和条件这两个模块彻底分开。上一个状态机