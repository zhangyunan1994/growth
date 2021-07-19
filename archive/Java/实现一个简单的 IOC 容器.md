# 实现一个简单的 IOC 容器 (一)

这篇文章主要讲一下如何使用 Java 实现一个简单的 IOC 容器，这里该系列的第一篇，要实现的内容的也相对简单，主要介绍一下 B 依赖 A 这种简单的关系是怎么实现的

![image-20210313163204421](../../img/ioc_node.png)



# Java 依赖注入标准 JSR-330 实现

我们常常使用的 Java DI 框架包括 Spring 和 Guice，在 Java 规范中也定义了对依赖注入的基本规范，其就是 JSR-330

标准对依赖注入的使用进行了定义, 但是对实现和配置未定义。包`javax.inject`对应该标准。具体实现依赖于各个框架。



## javax.injects

该包提供了如下5个注解（Inject、Qualifier、Named、Scope、Singleton）和1个接口（Provider）。



**@Inject**

标识某个类中，需要由注入器注入的类成员（被标识的成员称为”可注入的”）。

使用规则

可用于注解构造器、字段、方法这些类成员（对成员是静态与否、最好是public的）
每个类只能有一个构造器可以被标记可注入；空构造器可以不用@Inject注解。
可注入的字段不能为 final 的
可注入的方法不能为 abstract 的
注入器的依赖注入顺序

构造器 > 字段 > 方法
父类 > 子类
一个类的两个可注入字段或其他成员无注入顺序
另外的四个注解对依赖注入进一步进行配置。



**@Qualifier 和 @Named**

其中，@Qualifiery用于创建限定器。限定器是一个自定义的注解，可注解字段或方法的参数，用于限制可注入的依赖的类型。限定器注解必须被 @Qualifier 和 @Retention(RetentionPolicy.RUNTIME) 注解。



**@Scope 和 @Singleton**

其中，@Scope 用于创建作用域。作用域是一个自定义的注解，可注解构造器，用于要求注入器对注入的实例的创建方式。比如，是每次构造器被调用就创建一个依赖的实例，还是就创建一个依赖的实例然后重用。作用域注解必须被 @Scope 和 @Retention(RetentionPolicy.RUNTIME) 注解。

@Singleton 就是一个通过 @Scope 定义的作用域。



**Provider<T>**

Provider 作为另一种提供依赖的定义（有一种是 @Inject 注解），其实例提供 T 类型的实例。与 @Inject 注解相比，其还能：

返回的实例可以是多个
返回的实例可以是延迟返回的
返回的实例来自指定作用域内



# 实现思路

在这里在不考虑 AOP 的情况进行对 `@Inject` 和 `@Singleton` 进行实现，也就是只实现一个单例类型的依赖注入

![image-20210313164955462](../../img/ioc_node2.png)

在这个关系中，`Node` 作为一个单例对象，且不依赖于其他；`NodeB` 作为一个单例对象，并依赖于 `Node` 

在创建 NodeB 对象时，发现需要进行依赖注入，此时检测依赖对象 Node 是否创建，如果没有创建则创建 Node , 如果已经创建，则进行注入操作

**大致流程：**

1. NodeB 是否原来创建过，如果创建过直接返回
2. 获取 NodeB 的空参构造器和带有 **@Inject** 的构造器，如果无法找到对应的构造器则抛出异常
3. 在选择好的构造器中优先使用带有 **@Inject** 的构造器，如果没有使用空参数构造器
4. 将当前 NodeB 标记为**生成中**
5. 根据 NodeB 的构造器，获取构造器参数，如果是空参则直接生成，如果不是空参，则判断当前参数类型是不是被标记成生成中，如果被标记成**生成中**则抛出循环依赖异常，否则从第一步开始创建对应的对象，直到 NodeB 的所有的构造器依赖的参数都创建完成，进行有参构造器生成。
6. 假设上面生成的 NodeB 对应的对象实例为 baby
7. 获取 body 的所有的字段属性，并找出带有 **Inject.class** 注解的属性
8. 根据上面筛选出来的 Field, 获取 Field 对应的类型，如果对应类型已经生成，则直接赋值，如果对应的类型未生成，则从第一步开始生成指定的类型的实例对象
9. 获取 body 的所有非私有方法，并找出带有 **Inject.class** 注解的方法
10. 根据上面筛选出来的方法找到方法的参数，类似于构造器有参方法的步骤获取到所有的方法参数，并反射调用
11. 生成之后将 NodeB 的 **生成中** 标记去除，并加入已经生成结果中



# 具体实现

## 1. 定义个异常

这东西很简单，不做多解释了

```java
public class InjectException extends RuntimeException {
  public InjectException() {
    super();
  }

  public InjectException(String message, Throwable cause) {
    super(message, cause);
  }

  public InjectException(String message) {
    super(message);
  }

  public InjectException(Throwable cause) {
    super(cause);
  }

}
```



## 2. 定义一个容器 Injector

先确定一下最基本的 

- finalSingletonMap 用来保存已经生成好的实例
- processingInstances 用来保存处理中的类型
- getInstance 一个公共方法，用来获取对应的类型实例

```java
public class Injector {

  // 已经生成的单例
  private final Map<Class<?>, Object> finalSingletonMap = Collections.synchronizedMap(new HashMap<>());

  // 准备进行构造的类
  private final Set<Class<?>> processingInstances = Collections.synchronizedSet(new HashSet<>());

  public <T> T getInstance(Class<T> clazz) {
    return createNew(clazz);
  }
}
```



## 3. 构造器处理逻辑



### 3.1 获取构造器 createNew



这里 clazz 为我们要生成的实例的 class 类型

1. 判断类型是否已经生成，如果生成则直接返回对应的实例
2. 根据构造器生成对象实例

```java
Object o = finalSingletonMap.get(clazz);
if (o != null) {
  return (T) o;
}

ArrayList<Constructor<T>> constructors = new ArrayList<>();
T target = null;
// 获取对应的 class 的所有的构造器
for (Constructor<?> con : clazz.getDeclaredConstructors()) {
  if (con.isAnnotationPresent(Inject.class) && ReflectUtil.trySetAccessible(con)) {
    // 优先处理 Inject 构造器
    constructors.add(0, (Constructor<T>) con);
  } else if (con.getParameterCount() == 0 && ReflectUtil.trySetAccessible(con)) {
    // 再次获取无参构造器
    constructors.add((Constructor<T>) con);
  }
}

if (constructors.size() > 2) {
  // 如果大于 2 说明存在多个被 Inject 标记的构造器，此时无确定优先使用哪个
  throw new InjectException("无法确定使用哪个构造器进行构造 " + clazz.getCanonicalName());
}

if (constructors.size() == 0) {
  // 如果不存在可用的构造器，则无法构造
  throw new InjectException("无可用的构造器 " + clazz.getCanonicalName());
}
processingInstances.add(clazz); // 放入表示未完成的容器

target = createFromConstructor(constructors.get(0)); // 构造器注入

processingInstances.remove(clazz); // 从未完成的容器取出

// 处理所有的 field 注入
injectField(target);
injectMethod(target);
```



### 3.2 通过构造器创建对象

获取到构造器所需的所有参数的类型，并创建对应的类型，其创建步骤再次从 3.1 步骤开始

```java
private <T> T createFromConstructor(Constructor<T> con) {
  Object[] params = new Object[con.getParameterCount()];
  int i = 0;
  for (Parameter parameter : con.getParameters()) {
    if (processingInstances.contains(parameter.getType())) {
      throw new InjectException(
        String.format("循环依赖 class is %s", con.getDeclaringClass().getCanonicalName()));
    }
    Object param = createFromParameter(parameter);
    if (param == null) {
      throw new InjectException(String.format("无法创建构造器中的参数 %s of class %s",
                                              parameter.getName(), con.getDeclaringClass().getCanonicalName()));
    }
    params[i++] = param;
  }
  try {
    return con.newInstance(params);
  } catch (Exception e) {
    throw new InjectException("create instance from constructor error", e);
  }
}

@SuppressWarnings("unchecked")
private <T> T createFromParameter(Parameter parameter) {
  Class<?> clazz = parameter.getType();
  return (T) createNew(clazz);
}
```



## 4. 属性 Field 处理逻辑

如果细看的话，其实和构造器注入的逻辑是类似的，

1. 获取 body 的所有的字段属性，并找出带有 **Inject.class** 注解的属性
2. 根据上面筛选出来的 Field, 获取 Field 对应的类型，如果对应类型已经生成，则直接赋值，如果对应的类型未生成，则从 3.1 开始生成指定的类型的实例对象

```java
private <T> void injectField(T body) {
  List<Field> fields = new ArrayList<>();
  for (Field field : t.getClass().getDeclaredFields()) {
    if (field.isAnnotationPresent(Inject.class) && ReflectUtil.trySetAccessible(field)) {
      fields.add(field);
    }
  }
  for (Field field : fields) {
    Object f = createFromField(field);
    try {
      field.set(t, f);
    } catch (Exception e) {
      throw new InjectException(
        String.format("set field for %s@%s error", t.getClass().getCanonicalName(), field.getName()),
        e);
    }
  }
}

private <T> T createFromField(Field field) {
  Class<?> clazz = field.getType();
  return (T) createNew(clazz);
}
```



## 5. 处理注入方法

这个和构造器处理就大同小异了

```java
private <T> void injectMethod(T body) {
  List<Method> methods = new ArrayList<>();
  Method[] declaredMethods = body.getClass().getDeclaredMethods();
  for (Method declaredMethod : declaredMethods) {
    if (declaredMethod.getParameterCount() > 0
        && declaredMethod.isAnnotationPresent(Inject.class)
        && ReflectUtil.trySetAccessible(declaredMethod)) {
      methods.add(declaredMethod);
    }
  }

  int i;
  for (Method method : methods) {
    Object[] params = new Object[method.getParameterCount()];
    i = 0;
    for (Parameter parameter : method.getParameters()) {
      if (processingInstances.contains(parameter.getType())) {
        throw new InjectException(
          String.format("循环依赖 on method , the root class is %s", body.getClass().getCanonicalName()));
      }
      Object param = createFromParameter(parameter);
      params[i++] = param;
    }
    try {
      method.invoke(body, params);
    } catch (Exception e) {
      throw new InjectException("injectMethod ", e);
    }
  }
}
```



# 参考

- https://blog.csdn.net/u010278882/article/details/50773687



# 完整代码

Github：https://github.com/some-big-bugs/wheel-java-di

```java
package com.github.sbb.di;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import javax.inject.Inject;
import javax.inject.Singleton;

public class Injector {

  // 已经生成的单例
  private final Map<Class<?>, Object> finalSingletonMap = Collections.synchronizedMap(new HashMap<>());

  // 准备进行构造的类
  private final Set<Class<?>> processingInstances = Collections.synchronizedSet(new HashSet<>());

  /**
   * 获取对象
   *
   * @param clazz
   * @return
   */
  public <T> T getInstance(Class<T> clazz) {
    return createNew(clazz);
  }


  @SuppressWarnings("unchecked")
  private <T> T createNew(Class<T> clazz) {
    Object o = finalSingletonMap.get(clazz);
    if (o != null) {
      return (T) o;
    }

    ArrayList<Constructor<T>> constructors = new ArrayList<>();
    T target = null;
    for (Constructor<?> con : clazz.getDeclaredConstructors()) {
      // 优先处理 Inject 构造器
      if (con.isAnnotationPresent(Inject.class) && ReflectUtil.trySetAccessible(con)) {
        constructors.add(0, (Constructor<T>) con);
      } else if (con.getParameterCount() == 0 && ReflectUtil.trySetAccessible(con)) {
        constructors.add((Constructor<T>) con);
      }
    }

    if (constructors.size() > 2) {
      throw new InjectException("dupcated constructor for injection class " + clazz.getCanonicalName());
    }
    if (constructors.size() == 0) {
      throw new InjectException("no accessible constructor for injection class " + clazz.getCanonicalName());
    }
    processingInstances.add(clazz); // 放入表示未完成的容器

    target = createFromConstructor(constructors.get(0)); // 构造器注入

    injectField(target);
    injectMethod(target);

    processingInstances.remove(clazz); // 从未完成的容器取出

    boolean isSingleton = clazz.isAnnotationPresent(Singleton.class);
    if (isSingleton) {
      finalSingletonMap.put(clazz, target);
    }

    return target;
  }

  private <T> T createFromConstructor(Constructor<T> con) {
    Object[] params = new Object[con.getParameterCount()];
    int i = 0;
    for (Parameter parameter : con.getParameters()) {
      if (processingInstances.contains(parameter.getType())) {
        throw new InjectException(
            String.format("循环依赖 on constructor, class is %s", con.getDeclaringClass().getCanonicalName()));
      }
      Object param = createFromParameter(parameter);
      params[i++] = param;
    }
    try {
      return con.newInstance(params);
    } catch (Exception e) {
      throw new InjectException("create instance from constructor error", e);
    }
  }

  /**
   * 注入成员
   *
   * @param t
   */
  private <T> void injectField(T body) {
    List<Field> fields = new ArrayList<>();
    for (Field field : body.getClass().getDeclaredFields()) {
      if (field.isAnnotationPresent(Inject.class) && ReflectUtil.trySetAccessible(field)) {
        fields.add(field);
      }
    }
    for (Field field : fields) {
      Object f = createFromField(field);
      try {
        field.set(body, f);
      } catch (Exception e) {
        throw new InjectException(
            String.format("set field for %s@%s error", t.getClass().getCanonicalName(), field.getName()),
            e);
      }
    }
  }

  private <T> void injectMethod(T body) {
    List<Method> methods = new ArrayList<>();
    Method[] declaredMethods = body.getClass().getDeclaredMethods();
    for (Method declaredMethod : declaredMethods) {
      if (declaredMethod.getParameterCount() > 0
          && declaredMethod.isAnnotationPresent(Inject.class)
          && ReflectUtil.trySetAccessible(declaredMethod)) {
        methods.add(declaredMethod);
      }
    }

    int i;
    for (Method method : methods) {
      Object[] params = new Object[method.getParameterCount()];
      i = 0;
      for (Parameter parameter : method.getParameters()) {
        if (processingInstances.contains(parameter.getType())) {
          throw new InjectException(
              String.format("循环依赖 on method , the root class is %s", body.getClass().getCanonicalName()));
        }
        Object param = createFromParameter(parameter);
        params[i++] = param;
      }
      try {
        method.invoke(body, params);
      } catch (Exception e) {
        throw new InjectException("injectMethod ", e);
      }
    }
  }

  @SuppressWarnings("unchecked")
  private <T> T createFromParameter(Parameter parameter) {
    Class<?> clazz = parameter.getType();
    return (T) createNew(clazz);
  }

  @SuppressWarnings("unchecked")
  private <T> T createFromField(Field field) {
    Class<?> clazz = field.getType();
    return (T) createNew(clazz);
  }

}

```



















