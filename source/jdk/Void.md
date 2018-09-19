# Void
JDK1.8:

```java
package java.lang;

/**
 * The {@code Void} class is an uninstantiable(不可实例化) placeholder(占位符) class to hold a
 * reference to the {@code Class} object representing the Java keyword
 * void.
 *
 * @author  unascribed
 * @since   JDK1.1
 */
public final
class Void {

	/**
	 * The {@code Class} object representing the pseudo-type(伪类型) corresponding to
	 * the keyword {@code void}.
	 *
	 * 获得JVM中定义的原始类型void的Class对象
	 *
	 */
	@SuppressWarnings("unchecked")
	public static final Class<Void> TYPE = (Class<Void>) Class.getPrimitiveClass("void");

	/*
	 * The Void class cannot be instantiated(实例化).
	 */
	private Void() {}
}
```

## TEST