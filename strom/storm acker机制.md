<center><h2>storm acker机制</h2></center>

​      storm的acker机制保证消息完全处理完，storm允许用户在spout中发射一个新的源tuple时为其指定一个message id, 这个message id可以是任意的object对象。多个源tuple可以共用一个message id,表示这多个源tuple对用户来说是同一个消息单元。storm中记录级容错的意思是说，storm会告知用户每一个消息单元是否在**指定时间内**被完全处理了。那什么叫做完全处理呢，就是该message id绑定的源tuple及由该源tuple后续生成的tuple经过了topology中每一个应该到达的bolt的处理。

​	strom的topology中有一个系统级别的组件，叫做acker。这个acker的任务就是追踪从spout中的每一个message id绑定的若干tuple的处理路径，如果在用户设置的最大超时时间内这些tuple没有被完全处理，那么acker机会告知spout，该消息处理失败了，相反则会告知spout处理成功了。实际是上acker是利用了按位异或的思想来是想的：

   A xor B xor B xor A = 0;

  这里A,B在acker中就是对应tupleId,在spout中生成的tuple会制定一个64位的tuple id，这个tuple id会发送给acker以及对应要发射到的bolt中，当bolt接收到这个tuple id后也会将这个tuple id发送给acker。

看下图分析：

![storm_acker](/Users/zjlolife/Documents/mark_study/image/storm/storm_acker.png)

spout为新生成的tuple都绑定了messageId,后续bolt中生成的tuple也间接绑定了这个message id。

看看acker过程：

msgId: 0010 xor 1010 = 1000;

bot1: 0010 xor 1110 = 1100;再和上面异或  1100 xor 1000 = 0100;

bolt2: 1010 xor 1011 = 0001;再和上面异或 0001 xor 0100 = 0101;

bot3: 1110 xor 1011 = 0101;再和上面异或  0101 xor 0101 = 0;

最终acker结果就为0，则完全执行完。如果经过一段时间后，结果不为0，就重新代表该msgId没有完全执行完，会重复再执行，以确保至少执行一次。





