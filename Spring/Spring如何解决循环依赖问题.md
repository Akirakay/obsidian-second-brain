## 三级缓存
```Java
/**  
 * Cache of singleton objects: bean name --> bean instance 
 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);  

/**  
 * Cache of early singleton objects: bean name --> bean instance 
 */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);  

/**  
 * Cache of singleton factories: bean name --> ObjectFactory 
 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```

- **第一层缓存（singletonObjects）**：单例对象缓存池，已经实例化并且属性赋值，这里的对象是**成熟对象**；
- **第二层缓存（earlySingletonObjects）**：单例对象缓存池，已经实例化但尚未属性赋值，这里的对象是**半成品对象**；
- **第三层缓存（singletonFactories）**: 单例工厂的缓存

获取单例对象源码：
```Java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {  
    // 尝试从singletonObjects(一级缓存)中获取对象
    Object singletonObject = this.singletonObjects.get(beanName);  
    if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {  
	    // 若是获取不到而且对象在建立中，则尝试从earlySingletonObjects(二级缓存)中获取
        singletonObject = this.earlySingletonObjects.get(beanName);  
        if (singletonObject == null && allowEarlyReference) {  
            synchronized(this.singletonObjects) {  
                singletonObject = this.singletonObjects.get(beanName);  
                if (singletonObject == null) {  
                    singletonObject = this.earlySingletonObjects.get(beanName);  
                    if (singletonObject == null) {  
                        ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);  
                        if (singletonFactory != null) {  
		                    // 若是仍是获取不到而且容许从singletonFactories经过getObject获取，则经过singletonFactory.getObject()(三级缓存)获取
                            singletonObject = singletonFactory.getObject(); 
                            // 调整对象缓存位置 
                            this.earlySingletonObjects.put(beanName, singletonObject);  
                            this.singletonFactories.remove(beanName);  
                        }  
                    }  
                }  
            }  
        }  
    }  
  
    return singletonObject;  
}
```

简单理解为：
	对于A 依赖 B，B 依赖 A
	- A首先初始化，且将本身暴露到singletonFactories中
	- 尝试get(B)，由于B并未创建，于是B走创建流程
	- B初始化之后，尝试get(A)
	- 尝试一级缓存singletonObjects(确定没有，由于A还没初始化彻底)
	- 尝试二级缓存earlySingletonObjects（也没有）
	- 尝试三级缓存singletonFactories，因为A经过ObjectFactory将本身提早曝光了，因此B可以经过ObjectFactory.getObject拿到A对象(半成品)
	- B拿到A对象后顺利完成了初始化阶段一、二、三，彻底初始化以后将本身放入到一级缓存singletonObjects中
	- 此时返回A中，A此时能拿到B的对象顺利完成本身的初始化阶段二、三，最终A也完成了初始化，进去了一级缓存singletonObjects中，并且因为B拿到了A的对象引用，因此B如今hold住的A对象完成了初始化。

## Spring解决依赖循环的限制

1. 不能解决构造器注入循环依赖
Spring解决循环依赖主要是依赖三级缓存，但是**在调用构造方法之前还未将其放入三级缓存之中**
2. 不能解决prototype作用域对象
Spring不会缓存prototype对象

## 实际开发中依赖循环解决方式

- **生成代理对象产生的循环依赖**
这类循环依赖问题解决方法很多，主要有：
1. 使用@Lazy注解，延迟加载
2. 使用@DependsOn注解，指定加载先后关系
3. 修改文件名称，改变循环依赖类的加载顺序

- **使用@DependsOn产生的循环依赖**
这类循环依赖问题要找到@DependsOn注解循环依赖的地方，迫使它不循环依赖就可以解决问题。

- **多例循环依赖**
这类循环依赖问题可以通过把bean改成单例的解决。

- **构造器循环依赖**
这类循环依赖问题可以通过使用@Lazy注解解决。