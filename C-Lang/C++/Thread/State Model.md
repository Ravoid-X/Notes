## state_types.h
定义一个枚举来表示所有的状态
```
#pragma once

enum class StateType {
    Begin,
    Processing,
    //.....
};
```
## state_base.h
定义了每个状态必须遵守的接口
```
#pragma once

#include "message.h"
#include "state_types.h"
#include <string>
#include <optional> // 用于返回可能的状态切换请求

// 前向声明状态机类，以避免循环依赖
class StateMachine;

class StateBase {
public:
    // 构造函数，接收一个指向状态机上下文的指针
    explicit StateBase(StateMachine* stateMachine);
    virtual ~StateBase() = default;
    // 禁止拷贝和移动
    StateBase(const StateBase&) = delete;
    StateBase& operator=(const StateBase&) = delete;
    // 进入该状态时执行的操作
    virtual void onEnter() = 0;
    // 退出该状态时执行的操作
    virtual void onExit() = 0;
    // 处理从消息队列传来的消息
    // 如果需要切换状态，则返回新状态的类型；否则返回 nullopt
    virtual optional<StateType> handleMessage(const Message& msg) = 0;
    // 处理系统级错误
    virtual void handleError(const string& error) = 0;
protected:
    // 指向状态机上下文的指针
    StateMachine* m_stateMachine;
};
```
## state_base.cpp
```
#include "state_base.h"

StateBase::StateBase(StateMachine* stateMachine)
    : m_stateMachine(stateMachine) {}
```
## state_machine.h
```
#pragma once

#include "state_base.h"
#include "state_types.h"
#include "message.h"
#include <memory> // for unique_ptr

class StateMachine {
public:
    StateMachine();
    ~StateMachine() = default;
    // 从主循环调用，将消息分发给当前状态
    void handleMessage(const Message& msg);
    void handleError(const string& error);
private:
    void changeState(StateType newState);
    unique_ptr<StateBase> m_currentState;
    StateType m_currentStateType;
};
```
## state_machine.cpp
```
#include "state_machine.h"
#include "state_begin.h"
#include "state_processing.h" // 包含所有具体的状态头文件
#include <iostream>

StateMachine::StateMachine() : m_currentState(nullptr) {
    // 初始状态设置为 Begin
    changeState(StateType::Begin);
}

void StateMachine::handleMessage(const Message& msg) {
    if (m_currentState) {
        auto nextState = m_currentState->handleMessage(msg);
        if (nextState.has_value() && nextState.value() != m_currentStateType) {
            changeState(nextState.value());
        }
    }
}

void StateMachine::handleError(const string& error) {
    if (m_currentState) {
        m_currentState->handleError(error);
    }
}

void StateMachine::changeState(StateType newState) {
    if (m_currentState) {
        m_currentState->onExit();
    }
    //根据新状态类型创建对应的状态对象
    switch (newState) {
        case StateType::Begin:
            m_currentState = make_unique<StateBegin>(this);
            break;
        case StateType::Processing:
            m_currentState = make_unique<StateProcessing>(this);
            break;
        // case StateType::End:
        //     m_currentState = make_unique<StateEnd>(this);
        //     break;
        default:
            cerr << "错误：尝试切换到未知的状态类型！" << endl;
            //m_currentState = nullptr; // 或者切换到一个错误状态
            return;
    }
    m_currentStateType = newState;
    if (m_currentState) {
        m_currentState->onEnter();
    }
}
```
## state_begin.h
```
#pragma once

#include "state_base.h"

class StateBegin : public StateBase {
public:
    explicit StateBegin(StateMachine* stateMachine);

    void onEnter() override;
    void onExit() override;
    optional<StateType> handleMessage(const Message& msg) override;
    void handleError(const string& error) override;
};
```
## state_begin.cpp
```
#include "state_begin.h"
#include "state_machine.h"
#include <iostream>

StateBegin::StateBegin(StateMachine* stateMachine)
    : StateBase(stateMachine) {}

void StateBegin::onEnter() {

}

void StateBegin::onExit() {

}

optional<StateType> StateBegin::handleMessage(const Message& msg) {
    if (msg.id == 101) {
        return StateType::Processing;
    }
    return nullopt;
}

void StateBegin::handleError(const string& error) {

}
```
## main.cpp
```
#include "message.h"
#include "state_machine.h" 
#include <vector>
#include <thread>
#include <chrono>

int main() {
    StateMachine stateMachine;
    MessageQueue messageQueue;

    bool running = true;
    while (running) {
        Message received_msg;
        if (messageQueue.tryGetMessage(received_msg)) {
            if (received_msg.id == -1) {
                running = false;
                continue;
            }
            stateMachine.handleMessage(received_msg);
        } else {
            // 队列为空，短暂休眠，防止CPU占用过高
            this_thread::sleep_for(chrono::milliseconds(50));
        }
    }
    return 0;
}
```