# Looper Message Handler: Android的消息驱动架构

Android的“Looper Handler & Message机制”是Android framework的基础，可以说整个Android framework都是建立在这一基础机制上的。  

了解“Looper Handler & Message机制”，就需要认识该架构中的四个角色：[looper][Nc]、[handler][Nc]、[message queue][Nc]、[thread][Nc]。  

同时，通过一步步分析，我们会认识到，该架构的实现依赖两个更底层的实现：  
（1）Java中的thread local  
（2）Linux中的epoll机制  

在后面的叙述中，我需要逐渐阐述清楚如下几个问题:  
(1) Looper的native层原理  
(2) 消息驱动架构的体现  
(3) 异步是如何实现的  

[Nc]:./NamingConventions.md
