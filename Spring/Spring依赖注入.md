## 依赖注入的方式

### setter注入

```Java
@Service
public class UserService {
    // Setter 注入
    private UserDao userDao;

    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

缺点：
- 不可以注入final修饰的对象
- 注入对象可以被set方法修改

### 构造器注入

```Java
@Service
public class UserService {
    // constructor() 注入
    private UserDao userDao;

    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

优点：
- 可以注入final修饰的对象
- 注入对象不会被修改
- 注入对象完全初始化
- 更加通用

### 注解注入

```Java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

缺点：
- 无法注入final修饰的对象
- 只适用于IoC容器

## 推荐构造器注入的理由

1. 注入对象不可变：final修饰
2. 注入对象不为空：实例化对象会走我们自己实现的有参构造，如果依赖对象为空会报错
3. 注入对象完全初始化
4. 提前暴露循环依赖问题：注解注入只会在使用时暴露出来