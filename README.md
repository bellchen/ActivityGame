ActivityGame
============

an example of how to use state machine to build a game

[more information](http://studentdeng.github.io/blog/2014/11/05/ios-architecture2/)

iOS APP 架构漫谈（二）

状态机（State machine）

[原文](http://www.cocoachina.com/ios/20141113/10211.html)

上一篇《iOS APP 架构漫谈(一)》简单介绍了information flow的概念。这篇文章简单介绍另一个在编程中非常重要的思想或工具——状态机（State machine）。 对大多数计算机专业的家伙们来说，这应该是一门比较难学的课程，里面包含一大堆揪心的名字比如DFA，NFA，还有一大堆各种各样的数学符号,又是编译原理的基础。不过很遗憾，似乎在做完编译原理课程作业之后，很多人再也没有实现过或是用过状态机了。本文通过一个游戏demo来简单描述一下状态机在实践中的应用。demo code

背景

首先看下我们的使用场景，假如我们需要设计一套联网对战的小游戏。第一个难题可能是如何建立一个通道，让2个手机相互发送消息。这里我并不打算引入server端开发，希望只是通过客户端来实现这个逻辑，这里使用LeanCloud API来简化这个过程。这样我们可以暂时不考虑技术细节，直接站在业务角度去思考如何建立这个游戏。

业务场景–邀请

正式开始游戏之前，总会有一个邀请的环节。假如我们有2个用户，分别是Host，Guest。Host创建游戏，Guest加入游戏。游戏的整个流程和我们平时玩的对战游戏流程并没有多大不同。

2014_11_06_15_44_54.png

1. Host创建游戏，他就相当于进入一个等待队列里面。

2. Guest加入游戏，他从等待队列中找到一个匹配，比如Host。然后对Host发送join message

3. Host会收到很多join message。由于我们只是选择1vs1。这里假定Host同意Guest加入游戏。Host向Guest发送join confirm message

4. Guest收到join confirm message, 向Host发送Go消息，表示Guest已经进入游戏

5. Host收到Go消息。也进入游戏。

具体实现业务逻辑

现在的构想的逻辑只有5步，但其实还会包含很多逻辑，比如超时机制，重发机制。由于中间状态很多，还可能有我们没有想到过的问题。在面对这种复杂逻辑时，会通过状态机来帮助我们理顺逻辑。这时，我们脑中思考的业务其实是一个状态到一个状态的图。 如下

2014_11_06_10_30_32.png

上半部分是游戏的创建者，下半部分是游戏的加入者。

一开始，尽量简化模型，这里红色剪头表示我们的正确主流路线，黑色表现出错路线。也就是说，一旦错误，就回到原始Idle状态。

开始写代码

在想清楚所有逻辑，并考虑清楚正常路线和错误路线之后，就可以开始写代码了。为了方便，这里直接使用第三方的状态机框架TransitionKit。

定义State（HOST）

 TKState *idleState = [TKState stateWithName:@"idle"];
  TKState *waitingJoinState = [TKState stateWithName:@"waitingJoin"];
  TKState *waitingConfirmState = [TKState stateWithName:@"waitingConfirm"];
  TKState *goState = [TKState stateWithName:@"go"];
 
  [waitingConfirmState setDidEnterStateBlock:^(TKState *state, TKTransition *transition) {
      [selfWeak sendJoinConfirm];
  }];
 
  [goState setDidEnterStateBlock:^(TKState *state, TKTransition *transition) {
      NSLog(@"happy ending");
 
      [SVProgressHUD showSuccessWithStatus:@"ok"];
  }];
定义Event（HOST）

Event 是建立State到State的路径

 TKEvent *waitingJoinEvent = [TKEvent eventWithName:CUHostGameManagerWaitingJoinEvent
                           transitioningFromStates:@[idleState]
                                           toState:waitingJoinState];
 
  TKEvent *receiveInviteEvent = [TKEvent eventWithName:CUHostGameManagerReceiveInviteEvent
                               transitioningFromStates:@[waitingJoinState]
                                               toState:waitingConfirmState];
 
  TKEvent *receiveConfirmEvent = [TKEvent eventWithName:CUHostGameManagerReceiveConfirmEvent
                                transitioningFromStates:@[waitingConfirmState]
                                                toState:goState];
 
  TKEvent *disconnectedEvent = [TKEvent eventWithName:CUHostGameManagerDisconnectedEvent
                               transitioningFromStates:nil
                                              toState:idleState];
定义过程（HOST）

 - (void)startGame {
 
      NSAssert(self.session.peerId != nil, @"");
 
      //这里，如果不是idle，我们切换状态机到idle
      if (![self.stateMachine.currentState.name isEqual:@"idle"]) {
          [self fireEvent:CUHostGameManagerDisconnectedEvent userInfo:nil];
      }
 
      //这里调用LeanCloud 入队
      AVObject *waitingId = [AVObject objectWithClassName:@"waiting_join_Ids"];
      [waitingId setObject:self.session.peerId forKey:@"peerId"];
      [waitingId saveInBackgroundWithBlock:^(BOOL succeeded, NSError *error) {
          //enqueue 之后，进入waitingJoin状态
          [self fireEvent:CUHostGameManagerWaitingJoinEvent userInfo:nil];
      }];
  }
 
  - (void)sendJoinConfirm {
      //发送加入确认消息给Guest
      AVMessage *message = [AVMessage messageForPeerWithSession:self.session
                                                   toPeerId:self.peerId
                                                    payload:@"join_confirm"];
      [self.session sendMessage:message transient:YES];
  }
 
  - (void)session:(AVSession *)session didReceiveMessage:(AVMessage *)message
  {
      if ([message.payload isEqualToString:@"join"]) {
          //收到Join（邀请）之后，发送确认消息
          self.peerId = message.fromPeerId;
 
          //因为LeanCloud的API比较挫，watch 之后才能发送消息，但是我们不知道什么时候才watch成功。。。。
          //好在只是demo，我们只好用这种方式work around，延迟2s发送消息
          [NSObject cancelPreviousPerformRequestsWithTarget:self selector:@selector(sendInviteConfirmRequest:) object:nil];
          [self performSelector:@selector(sendInviteConfirmRequest:)
                      withObject:@[message.fromPeerId]
                      afterDelay:2.0f];
      }
      else if ([message.payload isEqualToString:@"go"]) {
          //收到go消息，流程结束
          [self fireEvent:CUHostGameManagerReceiveConfirmEvent userInfo:nil];
      }
  }
 
  - (void)sendInviteConfirmRequest:(NSArray *)watchPeerIds {
      [self.session watchPeerIds:watchPeerIds];
      [self fireEvent:CUHostGameManagerReceiveInviteEvent userInfo:nil];
  }
定义State（Guest）

 TKState *idleState = [TKState stateWithName:@"idle"];
  TKState *waitingReplyState = [TKState stateWithName:@"waitingReply"];
  TKState *goState = [TKState stateWithName:@"go"];
 
  [waitingReplyState setWillEnterStateBlock:^(TKState *state, TKTransition *transition) {
      [selfWeak searchingGames];
  }];
 
  [goState setDidEnterStateBlock:^(TKState *state, TKTransition *transition) {
      [selfWeak sendGo];
      NSLog(@"happy ending");
      [SVProgressHUD showSuccessWithStatus:@"ok"];
  }];
定义Event（Guest）

 TKEvent *searchingEvent = [TKEvent eventWithName:CUGestGameManagerSearchingEvent
                           transitioningFromStates:@[idleState]
                                           toState:waitingReplyState];
 
  TKEvent *receiveConfirmEvent = [TKEvent eventWithName:CUGestGameManagerReceiveConfirmEvent
                                transitioningFromStates:@[waitingReplyState]
                                                toState:goState];
 
  TKEvent *disconnectedEvent = [TKEvent eventWithName:CUGestGameManagerDisconnectedEvent
                              transitioningFromStates:nil
                                              toState:idleState];
定义过程（Guest）


- (void)joinGame {
 
  if (![self.stateMachine.currentState.name isEqual:@"idle"]) {
    [self fireEvent:CUGestGameManagerDisconnectedEvent userInfo:nil];
  }
 
  [self fireEvent:CUGestGameManagerSearchingEvent userInfo:nil];
}
 
- (void)searchingGames {
  AVQuery *query = [AVQuery queryWithClassName:@"waiting_join_Ids"];
  [query orderByDescending:@"updatedAt"];
  [query setLimit:1];
 
  [query findObjectsInBackgroundWithBlock:^(NSArray *objects, NSError *error) {
    NSMutableArray *installationIds = [[NSMutableArray alloc] init];
    for (AVObject *object in objects) {
      if ([object objectForKey:@"peerId"]) {
        [installationIds addObject:[object objectForKey:@"peerId"]];
      }
    }
 
    [self.session watchPeerIds:installationIds];
 
    [NSObject cancelPreviousPerformRequestsWithTarget:self selector:@selector(sendJoinRequest) object:nil];
    [self performSelector:@selector(sendJoinRequest)
               withObject:nil
               afterDelay:2.0f];
  }];
}
 
- (void)sendJoinRequest {
 
  for (NSString *item in self.session.watchedPeerIds) {
    AVMessage *message = [AVMessage messageForPeerWithSession:self.session
                                                     toPeerId:item
                                                      payload:@"join"];
    [self.session sendMessage:message transient:YES];
  }
}
 
- (void)sendGo{
  AVMessage *message = [AVMessage messageForPeerWithSession:self.session
                                                   toPeerId:self.otherPeerId
                                                    payload:@"go"];
  [self.session sendMessage:message transient:YES];
}
最后

state machine 是一个蛮厉害的锤子，只要是一个工具，就肯定会被滥用。。。state machine最大的好处是在于，方便我们思考清楚所有细节，主线，和错误流程。避免因为考虑不周全而产生的bug。结合之前的information flow的思路，会让我们的软件设计更加清楚。
