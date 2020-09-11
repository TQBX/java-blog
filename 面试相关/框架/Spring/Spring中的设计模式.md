- 工厂模式： spring中的BeanFactory就是简单工厂模式的体现，根据传入唯一的标识来获得bean对象；
- 单例模式： 提供了全局的访问点BeanFactory；
- 代理模式： AOP功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理。）
- 装饰器模式： 依赖注入就需要使用BeanWrapper；
- 观察者模式： spring中Observer模式常用的地方是listener的实现。如ApplicationListener。
- 策略模式： Bean的实例化的时候决定采用何种方式初始化bean实例（反射或者CGLIB动态字节码生成）

