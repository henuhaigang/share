# AOP 动态代理
---
## 一 JDK代理
DK动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理

    public class JDKProxy implements InvocationHandler {
    	//需要代理的目标对象
    	private Object targetObject;
    	// 构建代理类
    	private Object newProxy(Object targetObject) {
    		this.targetObject = targetObject;
    		return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), this);
    	}
    	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    		//模拟检查权限
    		checkPopedom();
    		//使用反射 调用代理类
    		Object value = method.invoke(proxy, args);
    		return value;
    	}
    	private void checkPopedom() {
    		System.out.println(".:检查权限 checkPopedom()!");
    	}
    }
        
## 二 CGLIB代理
动态生成一个要代理类的子类，子类重写要代理的类的所有不是final的方法。在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。它比使用java反射的JDK动态代理要快，cglib底层使用字节码处理框架ASM，来转换字节码并生成新的类。不鼓励直接使用ASM，因为它要求你必须对JVM内部结构包括class文件的格式和指令集都很熟悉，由于cglib要生成子类，所以对final类是无法进行代理的

    public class CGLIBProxy implements MethodInterceptor {
        private Object targetObject;
        private Object createProxyObject(Object targetObject) {
            this.targetObject = targetObject;
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(targetObject.getClass());
            enhancer.setCallback(this);
            Object proxyObject = enhancer.create();
            return proxyObject;
        }
    
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            Object object = null;
            if ("addUser".equals(method.getName())) {// 过滤方法
                checkPopedom();// 检查权限
            }
            object = method.invoke(targetObject, objects);
            return object;
        }
        private void checkPopedom() {
            System.out.println(".:检查权限 checkPopedom()!");
        }
    
    }
    
## 三 Spring proxy
### 3.1 代理方式选择规则(默认JDK P)
|:-|
|(1)如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP|
|(2)如果目标对象实现了接口，可以强制使用CGLIB实现AOP|
|(3)如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换|
### 3.2 CGLIB代理方式配置
#### 3.2.1 xml
在xml中配置如下标签    
    <aop:aspectj-autoproxy proxy-target-class="true">
#### 3.2.1 springboot
在application.properties或者application.yml去设置如下属性

    // application.properties
    spring.aop.proxy-target-class=true
    
    // application.yml
    spring：
        aop：
            proxy-target-class：true
            
## 四 字节码增强
### 4.1 ASM 
### 4.2 JAVASSIST   
### 4.3 CGLIB   
            

