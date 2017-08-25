---
title: spring AOP 获取目标方法参数
date: 2017-06-25 18:46:48
tags:
    - frame 
type: "categories" 
---

## 获取目标方法的信息

### 代码部分

访问目标方法最简单的做法是定义增强处理方法时，将第一个参数定义为JoinPoint类型，当该增强处理方法被调用时，该JoinPoint参数就代表了织入增强处理的连接点。JoinPoint里包含了如下几个常用的方法：

Object[] getArgs：返回目标方法的参数

Signature getSignature：返回目标方法的签名

Object getTarget：返回被织入增强处理的目标对象

Object getThis：返回AOP框架为目标对象生成的代理对象


中定义了Before、Around、AfterReturning 3个增强处理，并分别在3种增强处理中访问被织入增强处理的目标方法、目标方法的参数和被织入增强处理的目标对象等：

```
	/**
	 * 方法调用之前 aop:before
	 * 
	 * @param join
	 */
	public void befor(JoinPoint point) {
		 log.info(String.format("begin func: %s ,param: %s", point.getSignature().getDeclaringTypeName() + 
	                "." + point.getSignature().getName(),point.getArgs()));
	}

	/**
	 * around 调用前后
	 * o
	 * @param point
	 * @return
	 * @throws Throwable
	 */
	public Object paramProcess(ProceedingJoinPoint point) throws Throwable {
		// 访问目标方法的参数处理
		Object[] args = point.getArgs();
		// 用改变后的参数执行目标方法
		Object returnValue = point.proceed(args);
		System.out.println("@Around：执行目标方法之后...");
		System.out.println("@Around：被织入的目标对象为：" + point.getTarget());
		return "原返回值：" + returnValue + "，这是返回结果的后缀";
	}

	/**
	 * 方法调用之后处理
	 */
	public void end(JoinPoint point) {
		 log.info(String.format("end func: %s ,param: %s", point.getTarget(),point.getArgs()));
	}
```

###目标方法

```
package com.xxx.test
public class TargetFun{
	public void getData1(){
		System.out.println("我是目标方法.....");
	}
}

```

### 配置部分
```

<bean name="aopInter" class="com.xxx.aop.Interceptor" />
	<aop:config>
		<aop:aspect ref="aopInter">
			 <aop:around method="paramProcess" pointcut="execution(* com.xxx.test.*.getData*(..)) /> 
			<aop:before method="befor"
				pointcut="execution(* com.xxx.test.*.getData*(..))" />
			<aop:after-returning method="end"
				pointcut="execution(* com.xxx.test.*.getData*(..))"  /> 
		</aop:aspect>
	</aop:config>
	
```

### 测试方法

```
	ClassPathXmlApplicationContext context = null;
	TargetFun targetFun = null;

	@Before
	public void init() {
		String[] configs = { "classpath:spring/spring-platform-xxx.xml" };
		context = new ClassPathXmlApplicationContext(configs);
		targetFun = (TargetFun) context.getBean("test");
	}
	
		@Test
	public void testAopParam() {
		targetFun. getData1();
	}
```

