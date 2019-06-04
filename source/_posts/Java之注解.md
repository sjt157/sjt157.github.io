---
title: Java之注解
date: 2019-03-17 09:44:01
tags: Java
categories: Java
---

### 什么是注解？
用一个词就可以描述注解，那就是元数据，即一种描述数据的数据。所以，可以说注解就是源代码的元数据。比如，下面这段代码：

```java
@Override
public String toString() {
    return "This is String Representation of current object.";
}
```

上面的代码中，我重写了toString()方法并使用了@Override注解。但是，即使我不使用@Override注解标记代码，程序也能够正常执行。那么，该注解表示什么？这么写有什么好处吗？事实上，@Override告诉编译器这个方法是一个重写方法(描述方法的元数据)，如果父类中不存在该方法，编译器便会报错，提示该方法没有重写父类中的方法。如果我不小心拼写错误，例如将toString()写成了toStrring(){double r}，而且我也没有使用@Override注解，那程序依然能编译运行。但运行结果会和我期望的大不相同。现在我们了解了什么是注解，并且使用注解有助于阅读程序。

Annotation是一种应用于类、方法、参数、变量、构造器及包声明中的特殊修饰符。它是一种由JSR-175标准选择用来描述元数据的一种工具。

### 为什么要引入注解？
使用Annotation之前(甚至在使用之后)，XML被广泛的应用于描述元数据。不知何时开始一些应用开发人员和架构师发现XML的维护越来越糟糕了。他们希望使用一些和代码紧耦合的东西，而不是像XML那样和代码是松耦合的(在某些情况下甚至是完全分离的)代码描述。如果你在Google中搜索“XML vs. annotations”，会看到许多关于这个问题的辩论。最有趣的是XML配置其实就是为了分离代码和配置而引入的。上述两种观点可能会让你很疑惑，两者观点似乎构成了一种循环，但各有利弊。下面我们通过一个例子来理解这两者的区别。

假如你想为应用设置很多的常量或参数，这种情况下，XML是一个很好的选择，因为它不会同特定的代码相连。如果你想把某个方法声明为服务，那么使用Annotation会更好一些，因为这种情况下需要注解和方法紧密耦合起来，开发人员也必须认识到这点。

另一个很重要的因素是Annotation定义了一种标准的描述元数据的方式。在这之前，开发人员通常使用他们自己的方式定义元数据。例如，使用标记interfaces，注释，transient关键字等等。每个程序员按照自己的方式定义元数据，而不像Annotation这种标准的方式。

目前，许多框架将XML和Annotation两种方式结合使用，平衡两者之间的利弊。

通过实现ConstraintValidator完成自定义校验注解

### 注解分类
#### 按运行机制（注解存在于程序的那个阶段）将注解分为三类：
* 源码注解(只在源码存在)
* 编译注解(在class文件中也存在)
* 运行时注解(在运行阶段仍然起作用)

#### 按照来源来分的话，有如下三类：
* JDK自带的注解（Java目前只内置了三种标准注解：@Override、@Deprecated、@SuppressWarnings，以及四种元注解：@Target、@Retention、@Documented、@Inherited）
* 第三方的注解——这一类注解是我们接触最多和作用最大的一类
* 自定义注解——也可以看作是我们编写的注解，其他的都是他人编写注解

### 如何自定义一个注解？
编写一个校验手机号格式是否正确的校验器。背景：用户登录页面传入手机号作为账号，后端验证手机号格式是否正确

#### 代码
```java
package com.sjt.miaosha.validator;

import static java.lang.annotation.ElementType.ANNOTATION_TYPE;
import static java.lang.annotation.ElementType.CONSTRUCTOR;
import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import javax.validation.Constraint;
import javax.validation.Payload;

@Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER })
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {IsMobileValidator.class})
public @interface IsMobile {
	
	boolean required() default true;
	
	String message() default "手机号码格式错误";

	Class<?>[] groups() default { };

	Class<? extends Payload>[] payload() default { };
}

```
该自定义注解类中用到了四种元注解，最后一个@Constraint指定了校验类，也就是接下来的IsMobileValidator类。值得一提的是除了自定义的message、require属性外，下面的groups和payload也是必须添加的。

#### IsMobileValidator为自定义注解的校验类。
```java
package com.sjt.miaosha.validator;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

import org.apache.commons.lang3.StringUtils;

import com.sjt.miaosha.util.ValidatorUtil;

public class IsMobileValidator implements ConstraintValidator<IsMobile, String> {

	private boolean required = false;

	
	public void initialize(IsMobile constraintAnnotation) {
		required = constraintAnnotation.required();

	}

	
	public boolean isValid(String value, ConstraintValidatorContext context) {
		if (required) {
			return ValidatorUtil.isMobile(value);
		} else {
			if (StringUtils.isEmpty(value)) {
				return true;
			} else {
				return ValidatorUtil.isMobile(value);
			}
		}
	}

}

```

校验类需要实现ConstraintValidator接口。
接口使用了泛型，需要指定两个参数，第一个自定义注解类，第二个为需要校验的数据类型。
实现接口后要override两个方法，分别为initialize方法和isValid方法。其中initialize为初始化方法，可以在里面做一些初始化操作，isValid方法就是我们最终需要的校验方法了。可以在该方法中实现具体的校验步骤。本示例中进行了简单的手机号校验。

完成这几部分之后，一个简单的自定义校验注解就完成啦，不要忘记在使用的时候加上@Valid注解开启valid校验。


#### 那么如何获取在注解中定义的message信息呢？

在valid校验中，如果校验不通过，会产生BindException异常，捕捉到异常后可以获取到defaultMessage也就是自定义注解中定义的内容，具体实现如下：
```java
    BindException ex = (BindException)e;
    List<ObjectError> errors = ex.getAllErrors();
    ObjectError error = errors.get(0);
    String msg = error.getDefaultMessage();

```

### 总结
1. 自定义注解需要去手动实现两个文件：自定义注解类 + 注解校验器类
 
2. 自定义注解类：message() + groups() + payload() 必须；
 
3. 注解校验器类：继承 ConstraintValidator 类<注解类，注解参数类型> + 两个方法（initialize：初始化操作、isValid：逻辑处理）

### 参考
<http://www.importnew.com/10294.html>
<https://www.cnblogs.com/Qian123/p/5256084.html>
