# Spring Data REST 快速构建 restful api 应用



[TOC]

# 什么是Spring Data REST

> Spring Data REST是基于Spring Data的repository之上，可以把 repository 自动输出为REST资源，目前支持Spring Data JPA、Spring Data MongoDB、Spring Data Neo4j、Spring Data GemFire、Spring Data Cassandra的 repository 自动转换成REST服务。注意是自动。简单点说，Spring Data REST把我们需要编写的大量REST模版接口做了自动化实现.

# restful api

REST是一种设计风格(与具体的语言无关)，它的URL主体是资源，是个名词。而且也仅支持HTTP协议，规定了使用HTTP Method表达本次要做的动作，类型一般也不超过那四五种。这些动作表达了对资源仅有的几种转化方式。

常用的HTTP动词有下面五个（括号里是对应的SQL命令）。

- GET（SELECT）：从服务器取出资源（一项或多项）。
- POST（CREATE）：在服务器新建一个资源。
- PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。
- PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）。
- DELETE（DELETE）：从服务器删除资源。
- HEAD：获取资源的元数据。
- OPTIONS：获取信息，关于资源的哪些属性是客户端可以改变的。

# 实现

**Spring Boot 2.0.0.RELEASE**

## 添加依赖

**pom.xml**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

## 定义domain
```java
package com.zyndev.springdatarestdemo.domain;

import lombok.Data;

import javax.persistence.*;
import java.io.Serializable;
import java.util.Date;

/**
 * @author 张瑀楠 zyndev@gmail.com
 * @version 0.0.1
 */
@Data
@Entity
@Table(name = "tb_user")
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;            // 用户id
    private String userName;    // 用户名称
    private String password;    // 用户密码
    private Integer active;     // 是否可用
    private Date lastLoginTime; // 最后登录时间
    private Date createTime;    // 账户创建时间
    private Date updateTime;    // 最后更新时间
}

```

## 定义 Repository
```java
package com.zyndev.springdatarestdemo.controller;

import com.zyndev.springdatarestdemo.domain.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

/**
 * @author 张瑀楠 zyndev@gmail.com
 * @version 0.0.1
 */
@RepositoryRestResource(path="user")
public interface UserRepository extends JpaRepository<User, Long> {
}
```

## 配置
```yml
server:
  port: 8080

spring:
  jpa:
    hibernate:
      ddl-auto: update

```

> 通过设置`spring.jpa.hibernate.ddl-auto=update` 来自动创建表，如果你已经根据domain建好表，可忽略，配置中省略了数据库的配置，请根据情况自行添加

## 测试

启动项目:

**1. GET 访问 localhost:8080/user**
这里我已经添加了一条数据

```json
{
    "_embedded": {
        "users": [
            {
                "userName": "abc",
                "password": "abc",
                "active": 1,
                "lastLoginTime": null,
                "createTime": null,
                "updateTime": null,
                "_links": {
                    "self": {
                        "href": "http://localhost:8080/user/1"
                    },
                    "user": {
                        "href": "http://localhost:8080/user/1"
                    }
                }
            }
        ]
    },
    "_links": {
        "self": {
            "href": "http://localhost:8080/user{?page,size,sort}",
            "templated": true
        },
        "profile": {
            "href": "http://localhost:8080/profile/user"
        }
    },
    "page": {
        "size": 20,
        "totalElements": 1,
        "totalPages": 1,
        "number": 0
    }
}
```
**2. GET 访问 localhost:8080/user/1**

通过上面可以看出 `1` 是存在的

```json
{
    "userName": "abc",
    "password": "abc",
    "active": 1,
    "lastLoginTime": null,
    "createTime": null,
    "updateTime": null,
    "_links": {
        "self": {
            "href": "http://localhost:8080/user/1"
        },
        "user": {
            "href": "http://localhost:8080/user/1"
        }
    }
}
```

**3. GET 访问 localhost:8080/user/2**

因为 `2` 并不存在，这时返回 `404`

**4. POST localhost:8080/user  创建一个资源**

设置 `Content-Type=application/json`

**body:** 
```json
{
    "userName": "abcdfasdfe",
    "password": "abc",
    "active": 1,
    "lastLoginTime": null,
    "createTime": null,
    "updateTime": null
}
```

返回状态码 `201`

返回数据：
```json
{
    "userName": "abcdfasdfe",
    "password": "abc",
    "active": 1,
    "lastLoginTime": null,
    "createTime": null,
    "updateTime": null,
    "_links": {
        "self": {
            "href": "http://localhost:8080/user/4"
        },
        "user": {
            "href": "http://localhost:8080/user/4"
        }
    }
}
```

**5. 再次访问 GET 访问 localhost:8080/user**

这时可以看出 users 的数量为 2 说明已经创建成功
```json
{
    "_embedded": {
        "users": [
            {
                "userName": "abc",
                "password": "abc",
                "active": 1,
                "lastLoginTime": null,
                "createTime": null,
                "updateTime": null,
                "_links": {
                    "self": {
                        "href": "http://localhost:8080/user/1"
                    },
                    "user": {
                        "href": "http://localhost:8080/user/1"
                    }
                }
            },
            {
                "userName": "abcdfasdfe",
                "password": "abc",
                "active": 1,
                "lastLoginTime": null,
                "createTime": null,
                "updateTime": null,
                "_links": {
                    "self": {
                        "href": "http://localhost:8080/user/4"
                    },
                    "user": {
                        "href": "http://localhost:8080/user/4"
                    }
                }
            }
        ]
    },
    "_links": {
        "self": {
            "href": "http://localhost:8080/user{?page,size,sort}",
            "templated": true
        },
        "profile": {
            "href": "http://localhost:8080/profile/user"
        }
    },
    "page": {
        "size": 20,
        "totalElements": 2,
        "totalPages": 1,
        "number": 0
    }
}
```

**6. delete 访问 localhost:8080/user/1**

返回状态码： `204`

再次访问 GET 访问 localhost:8080/user 会发现 users 的数量已经为1，说明已经删除成功

> 可以使用 postman 测试，这里为了不贴图，就按上面的写了，希望理解

# 小功能

为了方便查看和测试api,可以集成 `hal browser`

在 pom 文件添加依赖即可
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-rest-hal-browser</artifactId>
</dependency>
```

重启项目并访问：`http://127.0.0.1:8080/browser/index.html#/`,效果如图


> 时间不早了，祝大家愚人节快乐，听过有人在加班，窃喜，如果喜欢请关注

