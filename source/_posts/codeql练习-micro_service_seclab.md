---
title: codeql实战练习-micro_service_seclab
tags:
  - 代码审计
  - myblog
---

不会的类要积极的去 CodeQL standard libraries 里面翻看。然后多积累，多记录。

# 构建数据库

项目地址 https://github.com/l4yn3/micro_service_seclab

`codeql database create ~/xxxxxx/micro-service-seclab-database  --language="java"  --command="mvn clean package -Dmaven.test.skip=true" --source-root=./micro-service-seclab/`

`codeql database create D:\codeqls\CodeQL-Practice --language="java" --source-root=D:\codeqls\micro_service_seclab --command="mvn clean package -Dmaven.test.skip=true"`

# 查看sql注入的点

>  `RemoteFlowSource`   : A data flow source of remote user input.

https://codeql.github.com/codeql-standard-libraries/java/semmle/code/java/dataflow/FlowSources.qll/type.FlowSources$RemoteFlowSource.html

比如下面这个

```java
@RequestMapping(value = "/one")    
public List<Student> one(@RequestParam(value = "username") String username) {    
    return indexLogic.getStudent(username);    
}
```

```java
 import java
 import semmle.code.java.dataflow.FlowSources
 import semmle.code.java.security.QueryInjection
 import DataFlow::PathGraph
 
 class VulConfig extends TaintTracking::Configuration {
      VulConfig() { this = "SqlInjectionConfig"}
     
     override predicate isSource(DataFlow::Node src) {
         src instanceof RemoteFlowSource
     }
     
     override predicate isSink(DataFlow::Node sink) {
         exists(Method method, MethodAccess call |
             method.hasName("query")
             and
             call.getMethod() = method and
             sink.asExpr() = call.getArgument(0)
         )
     }
 
 }
 
 from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
 where config.hasFlowPath(source, sink)
 select source.getNode(), source, sink, "source"
```

# 测试

在开始下面的这些之前。可以先来一个测试案例，然后看看官方的standard libraries 感受一下。

```java
import java
 
from Method method ,MethodAccess call
where call.getMethod() = method
select method,call,call.getArgument(0),call.getArgument(0).getType()
```

# 删除int类型参数

这个方法的参数类型是 `List<Long>`，不会存在注入漏洞。
这说明我们的规则里，对于 `List<Long>` ，甚至 `List<Integer>` 类型都会产生误报，source 误把这种类型的参数涵盖了。
我们需要采用 `isSanitizer` 来消除这种情况。

![image-20241129140751142](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241129140751142.png)

三个类对应的类型如下

PrimitiveType   > `boolean`, `byte`, `short`, `char`, `int`, `long`, `float`, and `double`.

BoxedType  > `Boolean`, `Byte`, `Short`, `Character`, `Integer`, `Long`, `Float`, and `Double`.

NumberType >  A (reflexive, transitive) subtype of `java.lang.Number`.

ParameterizedType 这个类，官方的一些解释。

> A parameterized type is an instantiation of a generic type(泛型类), where each formal type variable has been replaced with a type argument.
>
> For example, `List<Number>` is a parameterization of the generic type `List<E>`, where `E` is a type parameter.

```java
override predicate isSanitizer(DataFlow::Node node) {
    node.getType() instanceof PrimitiveType or
    node.getType() instanceof BoxedType or
    node.getType() instanceof NumberType or
    exists(ParameterizedType pt| 
        node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType
     )
}
```

# 解决漏报

我们发现，如下的SQL注入并没有被CodeQL捕捉到。

```java
public List<Student> getStudentWithOptional(Optional<String> username) {
        String sqlWithOptional = "select * from students where username like '%" + username.get() + "%'";
        //String sql = "select * from students where username like ?";
        return jdbcTemplate.query(sqlWithOptional, ROW_MAPPER);
}
```

假如 Optional 这种类型的使用没有在 CodeQL 的语法库里，我们需要强制让 `username` 流转到`username.get()`，这样 `username.get()` 就变得可控了。这样应该就能识别出这个注入漏洞了。

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph

predicate isTaintedString(Expr expSrc, Expr expDest) {
    exists(Method method, MethodAccess call, MethodAccess call1|
        expSrc = call1.getArgument(0) and expDest = call and call.getMethod() = method
        and method.hasName("get") and method.getDeclaringType().toString() = "Optional<String>"
        and call1.getArgument(0).getType().toString() = "Optional<String>"
        )
}

class VulConfig extends TaintTracking::Configuration {
     VulConfig() { this = "SqlInjectionConfig"}
    
    override predicate isSource(DataFlow::Node src) {
        src instanceof RemoteFlowSource
    }
    
    override predicate isSink(DataFlow::Node sink) {
        exists(Method method, MethodAccess call |
            method.hasName("query")
            and
            call.getMethod() = method and
            sink.asExpr() = call.getArgument(0)  // sink.asExpr() 是一个方法，用于将一个 sink 转换成一个表达式。这个方法通常用于在查询中使用 sink，因为查询需要将 sink 转换成表达式才能进行分析。
        )
    }

    override predicate isSanitizer(DataFlow::Node node) {
        node.getType() instanceof PrimitiveType or
        node.getType() instanceof BoxedType or
        node.getType() instanceof NumberType or
        exists(ParameterizedType pt| 
            node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType
         )
    }

    override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {
        isTaintedString(node1.asExpr(), node2.asExpr())
    }

}

from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

# fastjson

```java
 import java
 import semmle.code.java.dataflow.FlowSources
 import semmle.code.java.security.QueryInjection
 import DataFlow::PathGraph
 
 class FastjsonVulConfig extends TaintTracking::Configuration {
     FastjsonVulConfig() { this = "fastjson" }
     
     override predicate isSource(DataFlow::Node src) {
         src instanceof RemoteFlowSource
     }
     
     override predicate isSink(DataFlow::Node sink) {
         exists(Method method, MethodAccess call|
             method.hasName("parseObject")
             and
             call.getMethod() = method and
             sink.asExpr() = call.getArgument(0)
             )
     }
 }
 
 from FastjsonVulConfig fastjsonVul, DataFlow::PathNode source, DataFlow::PathNode sink
 where fastjsonVul.hasFlowPath(source, sink)
 select source.getNode(), source, sink, "source"
```

# SSRF

 RequestForgerySink 类。

> A data flow sink for server-side request forgery (SSRF) vulnerabilities.

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph
import semmle.code.java.security.RequestForgeryConfig

class SSRFVulConfig extends TaintTracking::Configuration {
		SSRFVulConfig() { this = "SSRFVulConfig" }
    
    override predicate isSource(DataFlow::Node src) {
        src instanceof RemoteFlowSource
    }
    
    override predicate isSink(DataFlow::Node sink) {
    sink instanceof RequestForgerySink
    }
 }
 from SSRFVulConfig ssrfVulConfig, DataFlow::PathNode source, DataFlow::PathNode sink
 where ssrfVulConfig.hasFlowPath(source, sink)
 select source.getNode(), source, sink, "source"
```

# ssrf 漏报处理1

## 漏报原因排查

但是其实ssrf 提供了5个路由都可以可以进行ssrf的，但是只排查出了3个。首先看看 two 这个路由 

```java
@RequestMapping(value = "/two")
public String Two(@RequestParam(value = "url") String imageUrl) {
    try {
        URL url = new URL(imageUrl);
        HttpResponse response = Request.Get(String.valueOf(url)).execute().returnResponse();
        return response.toString();
    } catch (IOException var1) {
        System.out.println(var1);
        return "Hello";
    }
}
```

`imageUrl→url = new URL(imageUrl)→String.valueOf(url)→Request.Get(String.valueOf(url))`

在 `two` 接口中的代码有问题的地方如下，用到了 `String.valueOf(url)`,正常情况下程序不会觉得 `String.valueOf()` 方法返回的仍然是污点。因此我们需要修改 Config 中的 `isAdditionalTaintStep` 方法，将 `java.net.URL` 和 `String.valueOf(url)` 绑定。

我们这里直接跟进RequestForgeryConfig 这个类看一看。

![image-20241130010138565](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241130010138565.png)

注意这个isAdditionalTaintStep。继续跟进propagatesTaint谓语。

![image-20241130010703439](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241130010703439.png)

## 处理方法1

我们这里直接修改 RequestForgery.qll  源码。主要是14到24行，还有 第 47 行。

```java
/** Provides classes to reason about server-side request forgery (SSRF) attacks. */

import java
import semmle.code.java.frameworks.Networking
import semmle.code.java.frameworks.ApacheHttp
import semmle.code.java.frameworks.spring.Spring
import semmle.code.java.frameworks.JaxWS
import semmle.code.java.frameworks.javase.Http
import semmle.code.java.dataflow.DataFlow
import semmle.code.java.frameworks.Properties
private import semmle.code.java.dataflow.StringPrefixes
private import semmle.code.java.dataflow.ExternalFlow

 class TypeStringLib extends RefType {
    TypeStringLib() { this.hasQualifiedName("java.lang", "String") }
  }

 class StringValue extends MethodAccess {
    StringValue(){
      this.getCallee().getDeclaringType() instanceof TypeStringLib and
      this.getCallee().hasName("valueOf")
    }
}

/**
 * A unit class for adding additional taint steps that are specific to server-side request forgery (SSRF) attacks.
 *
 * Extend this class to add additional taint steps to the SSRF query.
 */
class RequestForgeryAdditionalTaintStep extends Unit {
  /**
   * Holds if the step from `pred` to `succ` should be considered a taint
   * step for server-side request forgery.
   */
  abstract predicate propagatesTaint(DataFlow::Node pred, DataFlow::Node succ);
}

private class DefaultRequestForgeryAdditionalTaintStep extends RequestForgeryAdditionalTaintStep {
  override predicate propagatesTaint(DataFlow::Node pred, DataFlow::Node succ) {
    // propagate to a URI when its host is assigned to
    exists(UriCreation c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or
    // propagate to a URL when its host is assigned to
    exists(UrlConstructorCall c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or 
      //处理String.valueOf(URL)
    exists(StringValue c | c.getArgument(0) = pred.asExpr() | succ.asExpr() = c)
  }
}

private class TypePropertiesRequestForgeryAdditionalTaintStep extends RequestForgeryAdditionalTaintStep
{
  override predicate propagatesTaint(DataFlow::Node pred, DataFlow::Node succ) {
    exists(MethodAccess ma |
      // Properties props = new Properties();
      // props.setProperty("jdbcUrl", tainted);
      // Propagate tainted value to the qualifier `props`
      ma.getMethod() instanceof PropertiesSetPropertyMethod and
      ma.getArgument(0).(CompileTimeConstantExpr).getStringValue() = "jdbcUrl" and
      pred.asExpr() = ma.getArgument(1) and
      succ.asExpr() = ma.getQualifier()
    )
  }
}

/** A data flow sink for server-side request forgery (SSRF) vulnerabilities. */
abstract class RequestForgerySink extends DataFlow::Node { }

private class DefaultRequestForgerySink extends RequestForgerySink {
  DefaultRequestForgerySink() { sinkNode(this, "request-forgery") }
}

/** A sanitizer for request forgery vulnerabilities. */
abstract class RequestForgerySanitizer extends DataFlow::Node { }

private class PrimitiveSanitizer extends RequestForgerySanitizer {
  PrimitiveSanitizer() {
    this.getType() instanceof PrimitiveType or
    this.getType() instanceof BoxedType or
    this.getType() instanceof NumberType
  }
}

private class HostnameSanitizingPrefix extends InterestingPrefix {
  int offset;

  HostnameSanitizingPrefix() {
    // Matches strings that look like when prepended to untrusted input, they will restrict
    // the host or entity addressed: for example, anything containing `?` or `#`, or a slash that
    // doesn't appear to be a protocol specifier (e.g. `http://` is not sanitizing), or specifically
    // the string "/".
    exists(this.getStringValue().regexpFind("([?#]|[^?#:/\\\\][/\\\\])|^/$", 0, offset))
  }

  override int getOffset() { result = offset }
}

/**
 * A value that is the result of prepending a string that prevents any value from controlling the
 * host of a URL.
 */
private class HostnameSantizer extends RequestForgerySanitizer {
  HostnameSantizer() { this.asExpr() = any(HostnameSanitizingPrefix hsp).getAnAppendedExpression() }
}

```

## 处理方法2

或者不改写lib，直接改写 ql 查询语句。

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.dataflow.ExternalFlow
import DataFlow::PathGraph
import semmle.code.java.security.RequestForgeryConfig

class TypeStringLib extends RefType {
  TypeStringLib() { this.hasQualifiedName("java.lang", "String") }
}

class StringValue extends MethodAccess {
  StringValue(){
    this.getCallee().getDeclaringType() instanceof TypeStringLib and
    this.getCallee().hasName("valueOf")
  }
}

private class MyRequestForgeryAdditionalTaintStep extends RequestForgeryAdditionalTaintStep {
  override predicate propagatesTaint(DataFlow::Node pred, DataFlow::Node succ) {
    // propagate to a URI when its host is assigned to
    exists(UriCreation c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or
    // propagate to a URL when its host is assigned to
    exists(UrlConstructorCall c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or 
    //处理String.valueOf(URL)
    exists(StringValue c | c.getArgument(0) = pred.asExpr() | succ.asExpr() = c)
  }
}

class SSRFVulConfig extends TaintTracking::Configuration {
  SSRFVulConfig() { this = "first_modifySSRF" }

  override predicate isSource(DataFlow::Node source) {
      source instanceof RemoteFlowSource and
      // Exclude results of remote HTTP requests: fetching something else based on that result
      // is no worse than following a redirect returned by the remote server, and typically
      // we're requesting a resource via https which we trust to only send us to safe URLs.
      not source.asExpr().(MethodAccess).getCallee() instanceof UrlConnectionGetInputStreamMethod
    }

   override predicate isSink(DataFlow::Node sink) {
      sink instanceof RequestForgerySink
   }

   override predicate isAdditionalTaintStep(DataFlow::Node pred, DataFlow::Node succ) {
      any(RequestForgeryAdditionalTaintStep r).propagatesTaint(pred, succ)
    }


}
from SSRFVulConfig ssrfVulConfig, DataFlow::PathNode source, DataFlow::PathNode sink
where ssrfVulConfig.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

# ssrf 漏报处理2

还有一个漏的。

```java
@RequestMapping(value = "/three")
public String Three(@RequestParam(value = "url") String imageUrl) {
    try {
        URL url = new URL(imageUrl);
        OkHttpClient client = new OkHttpClient();
        com.squareup.okhttp.Request request = new com.squareup.okhttp.Request.Builder().get().url(url).build();
        Call call = client.newCall(request);
        Response response = call.execute();
        return response.toString();
    } catch (IOException var1) {
        System.out.println(var1);
        return "Hello";
    }
}
```

com.squareup.okhttp.Request request = new com.squareup.okhttp.Request.Builder().get().url(url).build();

这种请求相对更复杂，因此需要自行构造规则。在这一条 ql 的语句当中有两个关键的定位锚点，一个是`url(url)`，一个是 `build()`，`url()` 确定是否引入污点，`build()` 确定sink的位置。结合这两者，进行检测 ql 的构造。先看一下这个的语法树AST。

![image-20241130014216314](https://cocr-obsidian.oss-cn-beijing.aliyuncs.com/_typora_image/image-20241130014216314.png)

这种链式结构调用在语法树中是包含的关系，当获取到最外层的 MethodAccess 时，可以使用 `getAChildExpr()` 方法返回其子语句，使用 `getAChildExpr+()` 可以递归返回全部子语句。结合前面说到的两个关键定位锚点，进行如下代码构造

最终。效果。7-20 行 ，56 行

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.dataflow.ExternalFlow
import DataFlow::PathGraph
import semmle.code.java.security.RequestForgeryConfig

MethodAccess url(MethodAccess ma,DataFlow::Node node){
  exists( MethodAccess mc | mc = ma.getAChildExpr()| if mc.getCallee().hasName("url") and mc.getArgument(0) = node.asExpr() then result = mc else result = url(mc,node)
  )
}

MethodAccess m(DataFlow::Node node){
  exists(
      MethodAccess ma | ma.getCallee().hasName("build") and ma.getCallee().getDeclaringType().hasName("Builder") |result = url(ma,node)
  )
}

class TypeStringLib extends RefType {
  TypeStringLib() { this.hasQualifiedName("java.lang", "String") }
}

class StringValue extends MethodAccess {
  StringValue(){
    this.getCallee().getDeclaringType() instanceof TypeStringLib and
    this.getCallee().hasName("valueOf")
  }
}

private class MyRequestForgeryAdditionalTaintStep extends RequestForgeryAdditionalTaintStep {
  override predicate propagatesTaint(DataFlow::Node pred, DataFlow::Node succ) {
    // propagate to a URI when its host is assigned to
    exists(UriCreation c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or
    // propagate to a URL when its host is assigned to
    exists(UrlConstructorCall c | c.getHostArg() = pred.asExpr() | succ.asExpr() = c)
    or 
    //处理String.valueOf(URL)
    exists(StringValue c | c.getArgument(0) = pred.asExpr() | succ.asExpr() = c)
  }
}

class SSRFVulConfig extends TaintTracking::Configuration {
  SSRFVulConfig() { this = "first_modifySSRF" }

  override predicate isSource(DataFlow::Node source) {
      source instanceof RemoteFlowSource and
      // Exclude results of remote HTTP requests: fetching something else based on that result
      // is no worse than following a redirect returned by the remote server, and typically
      // we're requesting a resource via https which we trust to only send us to safe URLs.
      not source.asExpr().(MethodAccess).getCallee() instanceof UrlConnectionGetInputStreamMethod
    }

   override predicate isSink(DataFlow::Node sink) {
      sink instanceof RequestForgerySink or  
      //sink = URL对象
      exists (m(sink))
   }

   override predicate isAdditionalTaintStep(DataFlow::Node pred, DataFlow::Node succ) {
      any(RequestForgeryAdditionalTaintStep r).propagatesTaint(pred, succ)
    }


}
from SSRFVulConfig ssrfVulConfig, DataFlow::PathNode source, DataFlow::PathNode sink
where ssrfVulConfig.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```



# XXE

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.dataflow.ExternalFlow
import DataFlow::PathGraph

class XXEVulConfig extends TaintTracking::Configuration {
  XXEVulConfig(){
      this = "XXEVulConfig"
  }

  override predicate isSource(DataFlow::Node src) {
      src instanceof RemoteFlowSource
  }

  override predicate isSink(DataFlow::Node sink) {
      exists(Method method, MethodAccess call|
          method.hasName("parse") and
          call.getMethod() = method and
          sink.asExpr() = call.getArgument(0)
          )
  }
}

from XXEVulConfig xxeVulConfig, DataFlow::PathNode source, DataFlow::PathNode sink
where xxeVulConfig.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```



