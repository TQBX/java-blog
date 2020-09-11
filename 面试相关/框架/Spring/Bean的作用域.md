# Controller是单例还是多例，如何保证并发安全？

Controller默认是单例的，如果使用非静态的成员变量，可能会发生数据逻辑混乱，解决并发安全问题有如下几种方法 ：

- 尽量不要在controller中定义成员变量。
- 非要定义非静态成员变量，可以通过`@Scope("prototype")`，设置为原型模式。
- 使用ThreadLocal变量。

# SpringBean的作用域

- `singleton`：单例模式，Spring中的bean默认是单例的，当Spring创建ApplicationContext的时候，会初始化所有的单例非懒加载的实例，存入单例的缓存池中。
- `prototype`：原型模式，每次getBean的时候，会得到一个全新的实例，创建之后Spring不再对其进行管理。
- `request`：每次请求都新产生一个实例，与原型不同的是，创建之后，Spring会监听它。
- `session`：每次会话都新产生一个实例，与原型不同的是，创建之后，Spring会监听它。
- `global session`：全局的web域，类似于Servlet中的Application。

