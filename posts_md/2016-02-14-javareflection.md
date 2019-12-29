---
layout: post
title: Java 的反射大法
date: 2016-02-14 20:19:23
categories: main
---

## Java反射简介
---
>Java 反射是可以让我们在运行时获取类的函数、属性、父类、接口等 Class 内部信息的机制。通过反射还可以让我们在运行期实例化对象，调用方法，通过调用 get/set 方法获取／设置变量的值，即使方法或属性是私有的的也可以通过反射的形式调用，这种“看透 class”的能力被称为内省，这种能力在框架开发中尤为重要。 有些情况下，我们要使用的类在运行时才会确定，这个时候我们不能在编译期就使用它，因此只能通过反射的形式来使用在运行时才存在的类(该类符合某种特定的规范，例如 JDBC)，这是反射用得比较多的场景。还有一个比较常见的场景就是编译时我们对于类的内部信息不可知，必须得到运行时才能获取类的具体信息。比如 ORM 框架，在运行时才能够获取类中的各个属性，然后通过反射的形式获取其属性名和值，存入数据库。这也是反射比较经典应用场景之一。 

## 实例学习
---
为学习反射的使用，特意写了一个java工程来学习，为方便后面的说明，先把主要的类代码放出来，详细代码已经上传至[Github](https://github.com/rantianhua/JavaStudy)上。  

## Parent.java

~~~java
package reflection;

public abstract class Parent {
	//姓名
	public String mName;
	//性别
	public String mSex;
	
	public Parent (String name) {
		this("男",name);
	}
	
	public Parent (String sex,String name) {
		mSex = sex;
		mName = name;
	}
	
	abstract void greet();

	abstract void introduce();
	
	public void parentMethod() {
		System.out.println("this is a method in parent class");
	}

}  
~~~

## Study.java

~~~java
package reflection;

public interface Study {
	void study();
}
~~~

## Child.java

~~~java
package reflection;

public class Child extends Parent implements Study{
	
	private String mWork;
	private String mCompany;
	
	public int age;

	public Child(String name) {
		super(name);
	}
	
	public Child(String sex,String name) {
		super(sex,name);
	}
	
	public Child(String sex,String name,String work,String company) {
		super(sex,name);
		mWork = work;
		mCompany  = company;
	}

	private void setWorkAndCompany(String work,String company) {
		mWork = work;
		mCompany = company;
	}
	
	public String toString() {
		return "My name is " + mName + " and my work is " + mWork + " and I am working at " + mCompany;
	}
	
	public void study() {
		System.out.println("I am studying English.");
	}

	void greet() {
		// TODO Auto-generated method stub
		System.out.println("Hello, nice to meet you!");
	}

	void introduce() {
		// TODO Auto-generated method stub
		System.out.println("My name is " + mName);
	}
}
~~~

## 获取Class对象
---
若想检查一个类的信息，必修先获取这个类的Class对象。如果已经知道该类的类名，则可以通过`类名.class`方式获取；如果在编译期不知道目标类型，但是知道该类的完整路径，则可以通过`Class.forName("完整的类路径")`来获得；如果已经有了该类的一个实例**instance**，则可以用`instance.getClass()`方式获得该类的Class对象，具体如下所示：

~~~java
    /**
	 * 获取class对象
	 */
	public Class getClassObject() {
		System.out.println("\n==============getClassObject start==============" );
		//方法一：
		Class<?>  childClass = Child.class;
		System.out.println("get class object by class name : " + childClass.getName());
		//方法二：
		try {
			childClass = Class.forName("reflection.Child");
			System.out.println("get class object by the whole path : " + childClass.getName());
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
		//方法三：
		Child child = new Child("rth");
		childClass = child.getClass();
		System.out.println("get class object by the known object : " + childClass.getName());
		System.out.println("==============getClassObject end==============\n" );
		return childClass;
	}
~~~

## 通过Class对象获取一个该类的实例
---
要想获取该类的属性或者调用该类的方法，必须要有该类的一个实例（对象）。此过程要通过先前获得的Class对象得到该类的构造器对象，然后通过构造器对象来实例化一个该类的实例：

~~~java
    /**
	 * 通过class对象构造目标类型的对象
	 * @param mClass
	 * @return
	 */
	public Object getClassInstance(Class<?> mClass) {
		System.out.println("\n==============getClassInstance start==============" );
		Object ob = null;
		try {
			//通过class 对象获取构造器对象
			Constructor<?> constructor =  mClass.getConstructor(String.class,String.class,String.class,String.class);
			//设置constructor的Accessible
			constructor.setAccessible(true);
			//通过constructor对象创建Child对象
			ob = constructor.newInstance("男","rth","coder","Google");
		} catch (Exception e) {
			e.printStackTrace();
		}
		System.out.println("==============getClassInstance end==============\n" );
		return ob;
	}
~~~

## 获取当前类中定义的方法
---
Class对象的getDeclaredMethods()函数能获取当前类中所有的方法，与方法的可见性无关。而getDeclaredMethods(String name,Class<?>... parameterTypes)方法用来获取当前类中指定的方法，也与可见性无关。示例如下：

~~~java
    /**
	 * 通过反射获取指定对象中的方法
	 * @param child
	 */
	public void getDeclaredMethods(Child child) {
		System.out.println("\n==============getDeclaredMethods start==============" );
		//获取该类中所有的方法(public,private,protected,default)，但不包括从父类继承的方法
		Method[] methods = child.getClass().getDeclaredMethods();
		for(Method method : methods) {
			System.out.println("declared method name : " + method.getName());
		}
		try {
			//获取该类中特定的方法，与方法的可见性无关
			Method setWorkAndCompanyMethod = child.getClass().getDeclaredMethod("setWorkAndCompany"
					, String.class,String.class );
			//判断该方法是不是私有的
			if(Modifier.isPrivate(setWorkAndCompanyMethod.getModifiers())) {
				System.out.println(setWorkAndCompanyMethod.getName() + " is private");
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		System.out.println("==============getDeclaredMethods end==============\n" );
	}
~~~

## 获取当前类以及父类中的所有public方法
---
对应于getDeclaredMethods方法，还有一个类似的getMethods()方法，不同之处在于后者获取的是public型方法，而且包括父类中的方法。同样的，getDeclaredMethod(String name,Class<?>... parameterTypes)方法获取当前类或者父类中指定的方法。示例如下：

~~~java
    /**
	 * 获取当前类以及父类中的所有public方法
	 * @param child
	 */
	public void getMethods(Child child) {
		System.out.println("\n==============getMethods start==============" );
		//获取所有public方法
		Method[] methods = child.getClass().getMethods();
		for(Method method : methods) {
			System.out.println("method name : " + method.getName());
		}
		//获取制定的方法
		try {
			//只能获取共有方法，私有方法会抛出异常
			Method studyMethod = child.getClass().getMethod("study");
			//调用该public方法
			studyMethod.invoke(child);
			//尝试获取一个私有方法，将会抛出NoSuchMethodException错误
			Method setWorkAndCompanyMethod = child.getClass().getMethod("setWorkAndCompany"
					, String.class,String.class );
		} catch (Exception e) {
			System.out.println("getMethods出错："+e.getMessage());
		}
		System.out.println("==============getMethods end==============\n" );
	}
~~~

## 获取当前类中定义的属性
---
当前类中的属性可以通过getDeclaredFields()方法获取，若要获取当前类中指定的属性，通过getDeclaredField(String name)方法。这两种方法获取的属性都是与可见性无关的。

~~~java
    /**
	 * 获取当前类中定义的属性
	 * @param child
	 */
	public void getCurrentClassFields(Child child) {
		System.out.println("\n==============getCurrentClassFields start==============" );
		//获取当前类中所有属性，（public、private、protected）
		Field[] fields = child.getClass().getDeclaredFields();
		for(Field field : fields) {
			System.out.println("filed's name is : " + field.getName());
		}
		//获取指定的属性，与可见性无关
		try {
			Field mWork = child.getClass().getDeclaredField("mWork");
			//判断属性的可见性
			if(Modifier.isPrivate(mWork.getModifiers())) {
				System.out.println(mWork.getName() + " is private");
			}
		} catch (NoSuchFieldException | SecurityException|IllegalArgumentException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("==============getCurrentClassFields end==============\n" );
	}
~~~

## 获取当前类和父类中的公共属性
---
getFields()方法获取的是当前类以及父类中的所有公共属性。getField(String name)方法的意义同上，不再赘述。

~~~java
    /**
	 * 获取当前类和父类中所有的公有属性
	 * @param child
	 */
	public void getAllClassFields(Child child) {
		System.out.println("\n==============getAllClassFields start==============" );
		//获取当前类和父类所有的公共属性
		Field[] publicFields = child.getClass().getFields();
		for(Field field : publicFields) {
			System.out.println("public field name : " + field.getName());
		}
		//获取当前类和父类中指定的公共属性
		try {
			Field ageField = child.getClass().getField("age");
			//设置年龄
			ageField.setInt(child, 21);
			//打印年龄
			System.out.println("My age is " + ageField.getInt(child));
		} catch (NoSuchFieldException | SecurityException|IllegalArgumentException | IllegalAccessException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("==============getAllClassFields end==============\n" );
	}
~~~

## 获取当前类的父类
---
Class对象的getSuperClass()方法可以获得当前类的父类的Class对象：

~~~java
/**
	 * 获取父类
	 * @param child
	 */
	public void getParentClass(Child child) {
		System.out.println("\n==============getParentClass start==============" );
		Class<?> parentClass = child.getClass().getSuperclass();
		while(parentClass != null) {
			System.out.println("Child has a  super class is : " + parentClass.getName());
			parentClass = parentClass.getSuperclass();
		}
		System.out.println("==============getParentClass end==============\n" );
	}
~~~

## 获取当前类的接口
---
类似与getSuperClass()方法，getInterfaces()方法可以获得当前类实现的接口对象的Class对象：

~~~java
    /**
	 * 获取当前类的接口
	 * @param child
	 */
	public void getInterface(Child child) {
		System.out.println("\n==============getInterface start==============" );
		Class<?>[] interfaces = child.getClass().getInterfaces();
		for(Class<?> inf : interfaces) {
			System.out.println("child has implements a interface : " + inf.getName());
		}
		System.out.println("==============getInterface end==============\n" );
	}
~~~


