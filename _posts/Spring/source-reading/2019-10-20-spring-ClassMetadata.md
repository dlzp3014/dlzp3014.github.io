---
layout: post
title:  "Spring源码-ClassMetadata类元数据接口"
date:   2019-10-19 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

ClassMetadata定义了访问特定类的元数据接口，而不需要加载类。Spring框架提供的默认实现有两种，分别为：StandardClassMetadata(基于反射)和ClassMetadataReadingVisitor(基于ASM技术)




## ClassMetadata：类元数据接口

```java
/**
 * Interface that defines abstract metadata of a specific class,
 * in a form that does not require that class to be loaded yet.
 *
 * @since 2.5
 * @see StandardClassMetadata
 * @see org.springframework.core.type.classreading.MetadataReader#getClassMetadata()元数据读取器
 * @see AnnotationMetadata
 */
public interface ClassMetadata {

	/**
	 * Return the name of the underlying class.
	 */
	String getClassName();

	/**
	 * Return whether the underlying class represents an interface.
	 */
	boolean isInterface();

	/**
	 * Return whether the underlying class represents an annotation.
	 * @since 4.1
	 */
	boolean isAnnotation();

	/**
	 * Return whether the underlying class is marked as abstract.
	 */
	boolean isAbstract();

	/** 具体类
	 * Return whether the underlying class represents a concrete class,
	 * i.e. neither an interface nor an abstract class. 既不是接口也不是抽象类
	 */
	boolean isConcrete();

	/**
	 * Return whether the underlying class is marked as 'final'.
	 */
	boolean isFinal();

	/**
	 * Determine whether the underlying class is independent, i.e. whether
	 * it is a top-level class 顶层类 or a nested class (static inner class) that
	 * can be constructed independently from an enclosing class. 可以独立于封闭类构造
	 */
	boolean isIndependent();

	/**
	 * Return whether the underlying class is declared within an enclosing
	 * class (i.e. the underlying class is an inner/nested class or a 内部类或者嵌套类
	 * local class within a method).
	 * <p>If this method returns {@code false}, then the underlying
	 * class is a top-level class.
	 */
	boolean hasEnclosingClass();

	/**
	 * Return the name of the enclosing class of the underlying class,
	 * or {@code null} if the underlying class is a top-level class.
	 */
	@Nullable
	String getEnclosingClassName();

	/**超类
	 * Return whether the underlying class has a super class.
	 */
	boolean hasSuperClass();

	/**
	 * Return the name of the super class of the underlying class,
	 * or {@code null} if there is no super class defined.
	 */
	@Nullable
	String getSuperClassName();

	/** 接口
	 * Return the names of all interfaces that the underlying class
	 * implements, or an empty array if there are none.
	 */
	String[] getInterfaceNames();

	/** 类中声明的类
	 * Return the names of all classes declared as members of the class represented by
	 * this ClassMetadata object. This includes public, protected, default (package)
	 * access, and private classes私有类 and interfaces declared by the class 类声明的接口, but excludes 不包括继承的类和接口
	 * inherited classes and interfaces. An empty array is returned if no member classes
	 * or interfaces exist.接口
	 * @since 3.1
	 */
	String[] getMemberClassNames();

}

```

## StandardClassMetadata:标准类元数据

```java
/**
 * {@link ClassMetadata} implementation that uses standard reflection
 * to introspect a given {@code Class}.
 *
 * @author Juergen Hoeller
 * @since 2.5
 */
public class StandardClassMetadata implements ClassMetadata {

	private final Class<?> introspectedClass; 内省Class


	/**
	 * Create a new StandardClassMetadata wrapper for the given Class.
	 * @param introspectedClass the Class to introspect
	 */
	public StandardClassMetadata(Class<?> introspectedClass) {
		Assert.notNull(introspectedClass, "Class must not be null");
		this.introspectedClass = introspectedClass;
	}

	/**
	 * Return the underlying Class.
	 */
	public final Class<?> getIntrospectedClass() {
		return this.introspectedClass;
	}


	@Override
	public String getClassName() {
		return this.introspectedClass.getName();
	}

	@Override
	public boolean isInterface() {
		return this.introspectedClass.isInterface();
	}

	@Override
	public boolean isAnnotation() {
		return this.introspectedClass.isAnnotation();
	}

	@Override
	public boolean isAbstract() {
		return Modifier.isAbstract(this.introspectedClass.getModifiers());
	}

	具体类
	@Override
	public boolean isConcrete() {
		return !(isInterface() || isAbstract());
	}

	@Override
	public boolean isFinal() {
		return Modifier.isFinal(this.introspectedClass.getModifiers());
	}

	@Override
	public boolean isIndependent() {
		return (!hasEnclosingClass() ||
				(this.introspectedClass.getDeclaringClass() != null &&
						Modifier.isStatic(this.introspectedClass.getModifiers())));
	}

	@Override
	public boolean hasEnclosingClass() {
		return (this.introspectedClass.getEnclosingClass() != null);
	}

	@Override
	@Nullable
	public String getEnclosingClassName() {
		Class<?> enclosingClass = this.introspectedClass.getEnclosingClass();
		return (enclosingClass != null ? enclosingClass.getName() : null);
	}

	@Override
	public boolean hasSuperClass() {
		return (this.introspectedClass.getSuperclass() != null);
	}

	@Override
	@Nullable
	public String getSuperClassName() {
		Class<?> superClass = this.introspectedClass.getSuperclass();
		return (superClass != null ? superClass.getName() : null);
	}

	@Override
	public String[] getInterfaceNames() {
		Class<?>[] ifcs = this.introspectedClass.getInterfaces();
		String[] ifcNames = new String[ifcs.length];
		for (int i = 0; i < ifcs.length; i++) {
			ifcNames[i] = ifcs[i].getName();
		}
		return ifcNames;
	}

	@Override
	public String[] getMemberClassNames() {
		LinkedHashSet<String> memberClassNames = new LinkedHashSet<>(4);
		for (Class<?> nestedClass : this.introspectedClass.getDeclaredClasses()) { //内部声明的类
			memberClassNames.add(nestedClass.getName());
		}
		return StringUtils.toStringArray(memberClassNames);
	}

}

```

## ClassMetadataReadingVisitor:类元数据读取访问器

```java

/**
 * ASM class visitor which looks only for the class name and implemented types, ASM类访问器，只查找类名和实现的类型
 * exposing them through the {@link org.springframework.core.type.ClassMetadata} 通过ClassMetadata接口暴露
 * interface.
 *
 * @author Rod Johnson
 * @author Costin Leau
 * @author Mark Fisher
 * @author Ramnivas Laddad
 * @author Chris Beams
 * @since 2.5
 */
class ClassMetadataReadingVisitor extends ClassVisitor implements ClassMetadata {

	private String className = "";

	private boolean isInterface;

	private boolean isAnnotation;

	private boolean isAbstract;

	private boolean isFinal;

	@Nullable
	private String enclosingClassName;

	private boolean independentInnerClass;

	@Nullable
	private String superClassName;

	private String[] interfaces = new String[0];

	private Set<String> memberClassNames = new LinkedHashSet<>(4);


	public ClassMetadataReadingVisitor() {
		super(SpringAsmInfo.ASM_VERSION);
	}


	@Override
	public void visit(
			int version, int access, String name, String signature, @Nullable String supername, String[] interfaces) {

		this.className = ClassUtils.convertResourcePathToClassName(name); //转换资源路径为类名
		this.isInterface = ((access & Opcodes.ACC_INTERFACE) != 0);
		this.isAnnotation = ((access & Opcodes.ACC_ANNOTATION) != 0);
		this.isAbstract = ((access & Opcodes.ACC_ABSTRACT) != 0);
		this.isFinal = ((access & Opcodes.ACC_FINAL) != 0);
		if (supername != null && !this.isInterface) {
			this.superClassName = ClassUtils.convertResourcePathToClassName(supername);
		}
		this.interfaces = new String[interfaces.length];
		for (int i = 0; i < interfaces.length; i++) {
			this.interfaces[i] = ClassUtils.convertResourcePathToClassName(interfaces[i]);
		}
	}

	@Override
	public void visitOuterClass(String owner, String name, String desc) {
		this.enclosingClassName = ClassUtils.convertResourcePathToClassName(owner);
	}

	@Override
	public void visitInnerClass(String name, @Nullable String outerName, String innerName, int access) {
		if (outerName != null) {
			String fqName = ClassUtils.convertResourcePathToClassName(name);
			String fqOuterName = ClassUtils.convertResourcePathToClassName(outerName);
			if (this.className.equals(fqName)) {
				this.enclosingClassName = fqOuterName;
				this.independentInnerClass = ((access & Opcodes.ACC_STATIC) != 0);
			}
			else if (this.className.equals(fqOuterName)) {
				this.memberClassNames.add(fqName);
			}
		}
	}

	@Override
	public void visitSource(String source, String debug) {
		// no-op
	}

	@Override
	public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
		// no-op
		return new EmptyAnnotationVisitor();
	}

	@Override
	public void visitAttribute(Attribute attr) {
		// no-op
	}

	@Override
	public FieldVisitor visitField(int access, String name, String desc, String signature, Object value) {
		// no-op
		return new EmptyFieldVisitor();
	}

	@Override
	public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
		// no-op
		return new EmptyMethodVisitor();
	}

	@Override
	public void visitEnd() {
		// no-op
	}


	@Override
	public String getClassName() {
		return this.className;
	}

	@Override
	public boolean isInterface() {
		return this.isInterface;
	}

	@Override
	public boolean isAnnotation() {
		return this.isAnnotation;
	}

	@Override
	public boolean isAbstract() {
		return this.isAbstract;
	}

	@Override
	public boolean isConcrete() {
		return !(this.isInterface || this.isAbstract);
	}

	@Override
	public boolean isFinal() {
		return this.isFinal;
	}

	@Override
	public boolean isIndependent() {
		return (this.enclosingClassName == null || this.independentInnerClass);
	}

	@Override
	public boolean hasEnclosingClass() {
		return (this.enclosingClassName != null);
	}

	@Override
	@Nullable
	public String getEnclosingClassName() {
		return this.enclosingClassName;
	}

	@Override
	public boolean hasSuperClass() {
		return (this.superClassName != null);
	}

	@Override
	@Nullable
	public String getSuperClassName() {
		return this.superClassName;
	}

	@Override
	public String[] getInterfaceNames() {
		return this.interfaces;
	}

	@Override
	public String[] getMemberClassNames() {
		return StringUtils.toStringArray(this.memberClassNames);
	}

	//空的注解访问
	private static class EmptyAnnotationVisitor extends AnnotationVisitor {

		public EmptyAnnotationVisitor() {
			super(SpringAsmInfo.ASM_VERSION);
		}

		@Override
		public AnnotationVisitor visitAnnotation(String name, String desc) {
			return this;
		}

		@Override
		public AnnotationVisitor visitArray(String name) {
			return this;
		}
	}

	//空的方法访问
	private static class EmptyMethodVisitor extends MethodVisitor {

		public EmptyMethodVisitor() {
			super(SpringAsmInfo.ASM_VERSION);
		}
	}

	//属性访问
	private static class EmptyFieldVisitor extends FieldVisitor {

		public EmptyFieldVisitor() {
			super(SpringAsmInfo.ASM_VERSION);
		}
	}

}


```