---
title: 如何简单的创建一个 Unique Token
date: 2020-06-20 21:22:17
tags: Javascript Typescrpt
category: Tool
---

## 背景
在一个长连接的通信过程中，双方的通信不能一对一问答方式进行。
如下例： server 接收到链接建立的时间后立即发送了两个消息给 client, client 收到 service 发送的消息后立马回复了信息。
然后 server 接收到了这个信息， 但是它并不知道这个信息是 client 针对方才发送的消息的回复。

```typescript
// server
server.onConnection((connection) => {
  connection.sendMessage({
    data: 'hello'
  })
  connection.sendMessage({
    data: '你的状态好么？'
  })

connection.onMessage(message => {
  console.log('你是针对哪个消息的回复？', messgae)
})

})

```

```typescript
// client 
const connection = client.connect()

connection.onMessage((message) => {

  if(message.data === 'hello) {
    conection.sendMessage({
      data: 'i get it'
    })
  }
  if(message.data === '你的状态好么？) {
    conection.sendMessage({
      data: '还不错'
    })
  }
})
```

针对这样的情况我们应该怎么办呢？
- 给发送的每个消息添加一个唯一的 Token 作为一个唯一标识
- 一方收到消息后， 回复时将接收到的 Token 发送回去
- 发送放收到回复后根据收到的 id 来判断这个回复是针对哪一个消息的

那么问题来了，我们需要创一个 Token, 这个 Token 什么样的才比较合适？
- 唯一性（最重要的）

## 实现思路

1. 使用 uuid
2. 给一个 id = 0 自增

### uuid 有什么优劣
优： 能保证全局的唯一性

劣：
- 字符串占用空间大
- 生成的 id 很随机，不易读
- 做不了递增，不太能做到排序。revert 一个消息的时候很难通过 uuid 做到

### 自增 id 有什么优劣
优： 
- 占用空间小
- 生成有规律
- 可以排序
- 实现简单，不需要依赖系统

劣：
- 只能保证当前运行时的唯一

基于以上分析， 最后选择了自增 id 来实现一个 Unique Token

## 实现方法
自增 id 的 UToken 就很简单了， 直接给一个初始值 0 ，然后每次调用自增 1。

```typescript
class UToken {
  current = 0

  toString() {
    return this.current
  }

  valueOf() {
    return this.current
  }

  next() {
    return ++this.current
  }
}

const utoken = new UToken()

console.log(utoken) // 0
console.log(utoken.next()) // 1
console.log(utoken) // 1
```

功能是实现了， 但是运行时间过长，数值不断的变大，某一时刻肯定会超过 `Number.MAX_SAFE_INTEGER`， 此时 `Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2 将得到 true的结果，而这在数学上是错误的` 。 这个 UToken 就会失效。
这就跟大数计算（收精度限制）碰到的问题一样一样的。

#### 如何解决精度限制问题
将 number 拆分为数组存储，然后运算时递增，数组每一个单位值不超过 `Number.MAX_SAFE_INTEGER`，输出是将数组拼接成字符串输出

在本文中，我们会以链表的方式实现。链表的每一个节点的值均已进制数表示， 比如说本文所使用的表示52进制
链表的每一个节点记录了当前节点的值，上级一节点/下一节点的引用，以及相关方法

```typescript
class TokenNode {
  // 该节点的字符串值模板，让 TokenNode 不直接返回数字
  // 比如说 current = 1 => 对应的字符串为 B
  static template = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'

  current = 0

  // 前一个节点的引用
  prevNode: TokenNode

  // 下一个节点的引用
  nextNode: TokenNode

  // 节点类型
  constructor(public type?: string) {}

  /**
   * 节点自增
   * 如果增加后当前值大于模板长度，则前一个节点自增， 当前节点置0
   * 如果没有前置节点，则新建一个节点作为前置节点
   */
  add() {
    const ret = this.current + 1

    if (ret > TokenNode.template.length - 1) {
      if (this.prevNode) {
        this.prevNode.add()
      } else {
        this.prevNode = new TokenNode(this.type)
        this.prevNode.nextNode = this
      }
      this.current = 0
    } else {
      this.current = ret
    }
  }

  /**
   * 获取末端 tokenNode
   */
  private getLatestNode(): TokenNode {
    if (this.nextNode) {
      return this.getLatestNode()
    } 
    return this
  }

  /**
   * 整个 tokenNode 链自增， 及获取末端节点自增
   */
  fullAdd() {
    const latestNode = this.getLatestNode()
    latestNode.add()
    return this.getFullTokens()
  }

  /**
   * 获取当前 tokenNode 的字符串表示
   */
  getToken() {
    return TokenNode.template[this.current]
  }

  /**
   * 获取前置所有 tokenNode 的字符串表示
   */
  getPrevTokens(): string {
    if (this.prevNode) {
      return `${this.prevNode.getPrevTokens()}${this.prevNode.getToken()}`
    }
    return ''
  }

  /**
   * 获取后置所有 tokenNode 字符串表示
   */
  getNextTokens(): string {
    if (this.nextNode) {
      return `${this.nextNode.getToken()}${this.nextNode.getNextTokens()}`
    }
    return ''
  }

  /**
   * 获取整个 tokenNode 链的字符串表示
   */
  getFullTokens() {
    if(this.type === null || this.type === undefined) {
      return `${this.getPrevTokens()}${this.getToken()}${this.getNextTokens()}`
    }
    return `${this.type}__${this.getPrevTokens()}${this.getToken()}${this.getNextTokens()}`
  }
}

```

```typescript
class Token {
  private current: TokenNode
  constructor(private type?: string, private prefix: string = '', private suffix: string = '') {
    this.current = new TokenNode(this.type)
  }

  private toString() {
    return `${this.prefix}_${this.current.getFullTokens()}_${this.suffix}`
  }

  private valueOf() {
    return this.toString()
  }

  /**
   * genereate a token string
   */
  generate() {
    return this.current.fullAdd()
  }
}
```

然后我们就可以生成一个满足条件的 Unique Token

```typescript
// test
import {
  Token
} from '@nannanbug/unique-token'

describe('utils/token', () => {
  let token: Token | undefined
  beforeEach(() => {
    token = new Token()
  })

  afterAll(() => {
    token = undefined
  })

  test('new Token', () => {
    expect(`${token}`).toBe('_A_')

    token?.generate()
    expect(`${token}`).toBe('_B_')

    for(let i = 0; i < 300; i++) {
      token?.generate()
    }
    expect(`${token}`).toBe('_Ep_')

  })
})
```

```typescript
// 实际使用
import { Token } from '@nananbug/unique-token'

const VNodeID = new Token('vnode')

const VNodeView = {
  id: `${VNodeID}`,
  type: 'View',
  props: {}
}
// {
//   id: '_vnode__A_',
//   type: 'View',
//   props: {}
// }

VNodeID.generate()
const VNodeImage = {
  id: `${VNodeID}`,
  type: 'Image',
  props: {}
}
// {
//   id: '_vnode__B_',
//   type: 'Image',
//   props: {}
// }

```

## 最后
相应的包可查看 [@nannanbug/unique-token](https://github.com/looading/unique-token)

End of the battle