### I/O复用

&emsp;&emsp;当客户端或服务端程序需要同时处理多个socket时，需要用到I/O复用技术，最常见的是服务端程序同时处理监听socket和连接socket。  
&emsp;&emsp;Linux提供了select、poll和epoll来实现I/O复用，前两者都工作在LT模式下，epoll有LT和ET两种工作模式。

#### LT和ET模式

&emsp;&emsp;epoll有LT(Level Trigger，电平触发)和ET(Edge Trigger，边缘触发)两种模式。select和poll都工作在LT模式。  
&emsp;&emsp;采用LT模式工作的文件描述符，当epoll_wait检测到有事件发生通知该应用程序后，应用程序不用立即处理，当系统下一次调用epoll_wait时，会再次通知应用程序，直至事件被处理。采用ET模式工作的文件描述符，当epoll_wait检测到有事件发生通知该应用程序后，应用程序必须立即处理，后续的epoll_wait调用不会再通知该事件。


#### EPOLLONESHOT事件

&emsp;&emsp;即使使用ET模式，一个socket上的某个事件也还是可能被多次触发，这在并发程序中会导致问题，一个线程接收到数据后正在处理，处理过程中有新的数据可读，这时EPOLLIN事件再次被触发，另一个线程被唤醒读取新的数据，这样导致了两个线程处理同一个socket。  
&emsp;&emsp;对注册了EPOLLONESHOT事件的文件描述符，系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次，这样保证了一个线程处理该socket时，其他线程无法操作该socket。反过来，一旦该线程处理完注册了EPOLLONESHOT事件的socket，应该立即重置EPOLLONESHOT事件，确保这个socket下一次可读时，能够被其他线程处理。

#### 三种方式实现I/O复用的区别

<table style="text-align:center;border:1px solid">
    <caption style="margin:0px 0px 5px 0px;font-weight:bold;font-size:18px">select、poll和epoll区别</caption>
    <tr>
        <th style="width:10%">系统调用</th>
        <th style="width:30%">select</th>
        <th style="width:30%">poll</th>
        <th style="width:30%">epoll</th>
    </tr>
    <tr>
        <td>事件集合</td>
        <td>&emsp;&emsp;用户通过三个参数分别传入可读、可写和异常等事件，内核通过对这些事件的在线修改来反馈其中的就绪事件，这是使得用户每次调用select都要重置这三个参数。</td>
        <td>&emsp;&emsp;统一处理所有事件类型，因此只需一个事件集参数。用户通过pollfd.events传入感兴趣的事件，内核通过修改poll.revents反馈其中就绪的事件。</td>
        <td>&emsp;&emsp;内核通过一个事件表统一管理用户感兴趣的所有事件，因此每次调用epoll_wait时无需反复传入感兴趣的事件，epoll_wait系统调用的参数events仅用来反馈就绪的事件。</td>
    </tr>
    <tr>
        <td>应用程序索引就绪文件描述符时间复杂度</td>
        <td>O(n)</td>
        <td>O(n)</td>
        <td>O(1)</td>
    </tr>
    <tr>
        <td>最大支持文件描述数量</td>
        <td>一般有最大值限制</td>
        <td>65535</td>
        <td>65535</td>
    </tr>
    <tr>
        <td>工作模式</td>
        <td>LT</td>
        <td>LT</td>
        <td>支持ET高效模式</td>
    </tr>
    <tr>
        <td>内核实现</td>
        <td>&emsp;&emsp;采用轮询方式检测就绪事件，算法复杂度为O(n)。</td>
        <td>&emsp;&emsp;采用轮询方式检测就绪事件，算法复杂度为O(n)。</td>
        <td>&emsp;&emsp;采用回调方式检测就绪事件，算法复杂度为O(1)。</td>
    </tr>
</table>