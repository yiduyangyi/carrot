---
title: java 8新特性小记
tags: ["java8"]
categories: ["java", "语言基础"]
icon: fa-handshake-o
---

# java 8 新特性

## 语言层面

### stream
stream api

	int main()  {

	}


### lambda表达式


### 函数接口
@FunctionalInterface，用来标注接口为函数接口，只允许有一个方法，但可以有默认方法和静态方法。该注解主要用于克服函数接口的脆弱性。


### 接口的默认方法和静态方法

### 方法引用

	import java.util.function.Supplier;

	/**
	 * Created by yangjinfeng on 2017/3/4.
	 */
	public class Bicycle {

	    public static Bicycle create(final Supplier<Bicycle> supplier) {
	        return supplier.get();
	    }

	    public static void producer(final Bicycle bicycle) {
	        System.out.println("bicycle producer " + bicycle.toString());
	    }

	    public void name(final Bicycle bicycle) {
	        System.out.println("bicycle name " + bicycle.toString());
	    }

	    public void repair() {
	        System.out.println("bicycle repair " + this.toString());
	    }
	}


1.构造方法引用，语法是：Class::new ，对于泛型来说语法是：Class<T >::new，请注意构造方法没有参数

	Bicycle bicycle = Bicycle.create(Bicycle::new);
	List<Bicycle> bicycles = Arrays.asList(bicycle);
2.静态方法引用，语法是：Class::static_method，请注意这个静态方法只支持一个类型为Bicycle的参数。

    bicycles.forEach(Bicycle::producer);
3.类的非静态，无参方法引用，语法是：Class::method，请注意方法没有参数。

    bicycles.forEach(Bicycle::repair);
4.类实例的单个参数的方法引用，语法是：instance::method，请注意只接受Bicycle类型的一个参数。

	bicycles.forEach(bicycle::name);


### 增强的类型推导

### 参数名称
java 8 开始支持将参数名称保存到字节码中，可通过反射获取。

	public class Application {

	    public static void main(String[] args) throws NoSuchMethodException {

	        Method method = Application.class.getMethod("main", String[].class);
	        for (Parameter parameter : method.getParameters()) {
	            System.out.println("parameter " + parameter.getName());
	        }
	    }
	}

### 重复注解
自从Java 5支持注释以来，注释变得特别受欢迎因而被广泛使用。但是有一个限制，同一个地方的不能使用同一个注释超过一次。
Java 8打破了这个规则，引入了重复注释，允许相同注释在声明使用的时候重复使用超过一次。重复注释本身需要被@Repeatable注释。
实际上，他不是一个语言上的改变，只是编译器层面的改动，技术层面仍然是一样的。

### 注解扩展
Java 8扩展了注解可以使用的范围，现在我们几乎可以在所有的地方：局部变量、泛型、超类和接口实现、甚至是方法的Exception声明。
Java 8 新增加了两个注解的程序元素类型ElementType.TYPE_USE 和 ElementType.TYPE_PARAMETER ，这两个新类型描述了可以使用注解的新场合。注解处理API（Annotation Processing API）也做了一些细微的改动，来识别这些新添加的注解类型。


## 类库层面

### Optional
避免NullPointerException而引入的Optional容器

### Collection & Stream
增加stream相关函数式编程API

### 日期时间API JSR310

### 并行数组


## 工具层面

### jdeps


# 参考资料
[java 8特性——终极手册](http://ifeve.com/java-8-features-tutorial/)

[官方文档——what's new](http://www.oracle.com/technetwork/java/javase/8-whats-new-2157071.html)
