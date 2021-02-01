## LinkedTransferQueue概述

LinkedTransferQueue是由链表组成的无界**TransferQueue**，相对于其他阻塞队列，多了tryTransfer和transfer方法。

TransferQueue：<u>生产者会一直阻塞直到所添加到队列的元素被某一个消费者所消费（不仅仅是添加到队列里就完事）。</u>新添加的transfer方法用来实现这种约束。顾名思义，阻塞就是发生在元素从一个线程transfer到另一个线程的过程中，它有效地实现了元素在线程之间的传递（以建立Java内存模型中的happens-before关系的方式）。

