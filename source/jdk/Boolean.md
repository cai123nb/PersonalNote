# Boolean

JDK1.8:

```java
package java.lang;

/**
 * The Boolean class wraps a value of the primitive type
 * {@code boolean} in an object. An object of type
 * {@code Boolean} contains a single field whose type is
 * {@code boolean}.
 * <p>
 * In addition, this class provides many methods for
 * converting a {@code boolean} to a {@code String} and a
 * {@code String} to a {@code boolean}, as well as other
 * constants and methods useful when dealing with a
 * {@code boolean}.
 *
 * @author  Arthur van Hoff
 * @since   JDK1.0
 */
 public final class Boolean implements java.io.Serializable,
                                      Comparable<Boolean>
{
   /**
    * The {@code Boolean} object corresponding to the primitive
    * value {@code true}.
    * 预构造的TRUE常量对象
    */
   public static final Boolean TRUE = new Boolean(true);

   /**
    * The {@code Boolean} object corresponding to the primitive
    * value {@code false}.
    * 预构造的FALSE常量对象
    */
   public static final Boolean FALSE = new Boolean(false);

   /**
    * The Class object representing the primitive type boolean.
    *
    * JVM机中原始类型boolean对应的Class对象
    *
    * @since   JDK1.1
    */
   @SuppressWarnings("unchecked")
   public static final Class<Boolean> TYPE = (Class<Boolean>) Class.getPrimitiveClass("boolean");

   /**
    * The value of the Boolean.
    *
    * Boolean对象中封装的boolean值（final，不可变的）
    *
    * @serial
    */
   private final boolean value;

   /** use serialVersionUID from JDK 1.0.2 for interoperability(互通性,互操作性) */
   private static final long serialVersionUID = -3665804199014368530L;

   /**
    * Allocates a {@code Boolean} object representing the
    * {@code value} argument.
    *
    * <p><b>Note: It is rarely(很少) appropriate(合适的) to use this constructor.
    * Unless a <i>new</i> instance is required, the static factory
    * {@link #valueOf(boolean)} is generally a better choice. It is
    * likely to yield(得到) significantly better space and time performance.</b>
    *
    * @param   value   the value of the {@code Boolean}.
    */
   public Boolean(boolean value) {
      this.value = value;
   }

   /**
    * Allocates a {@code Boolean} object representing the value
    * {@code true} if the string argument is not {@code null}
    * and is equal, ignoring case, to the string {@code "true"}.
    * Otherwise, allocate a {@code Boolean} object representing the
    * value {@code false}. Examples:<p>
    * {@code new Boolean("True")} produces a {@code Boolean} object
    * that represents {@code true}.<br>
    * {@code new Boolean("yes")} produces a {@code Boolean} object
    * that represents {@code false}.
    *
    * 传递一个String对象,如果为空或者不为true(忽视大小写),则返回false.
    *
    * @param   s   the string to be converted to a {@code Boolean}.
    */
   public Boolean(String s) {
      this(parseBoolean(s));
   }

   /**
    * Parses the string argument as a boolean.  The {@code boolean}
    * returned represents the value {@code true} if the string argument
    * is not {@code null} and is equal, ignoring case, to the string
    * {@code "true"}. <p>
    * Example: {@code Boolean.parseBoolean("True")} returns {@code true}.<br>
    * Example: {@code Boolean.parseBoolean("yes")} returns {@code false}.
    *
    * 传递一个String对象,如果不为空且和true相同(无视大小写),返回true原始类型.
    * 否则返回false.
    *
    * @param      s   the {@code String} containing the boolean
    *                 representation to be parsed
    * @return     the boolean represented by the string argument
    * @since 1.5
    */
   public static boolean parseBoolean(String s) {
      return ((s != null) && s.equalsIgnoreCase("true"));
   }

   /**
    * Returns the value of this {@code Boolean} object as a boolean
    * primitive.
    *
    * 返回原始类型boolean变量
    *
    * @return  the primitive {@code boolean} value of this object.
    */
   public boolean booleanValue() {
      return value;
   }

   /**
    * Returns a {@code Boolean} instance representing the specified
    * {@code boolean} value.  If the specified {@code boolean} value
    * is {@code true}, this method returns {@code Boolean.TRUE};
    * if it is {@code false}, this method returns {@code Boolean.FALSE}.
    * If a new {@code Boolean} instance is not required, this method
    * should generally be used in preference to the constructor
    * {@link #Boolean(boolean)}, as this method is likely to yield
    * significantly better space and time performance.
    *
    * 该方法的优先级应该大于构造函数,通过返回预构造的TRUE对象和FALSE对象
    * 可以得到更好的空间性能和时间性能.
    *
    * @param  b a boolean value.
    * @return a {@code Boolean} instance representing {@code b}.
    * @since  1.4
    */
   public static Boolean valueOf(boolean b) {
      return (b ? TRUE : FALSE);
   }

   /**
    * Returns a {@code Boolean} with a value represented by the
    * specified string.  The {@code Boolean} returned represents a
    * true value if the string argument is not {@code null}
    * and is equal, ignoring case, to the string {@code "true"}.
    *
    * 转换String为TRUE或者FALSE的对象, 原理和parseBoolean()函数一致
    * 同样返回预构造好的TRUE和FALSE对象.
    *
    * @param   s   a string.
    * @return  the {@code Boolean} value represented by the string.
    */
   public static Boolean valueOf(String s) {
      return parseBoolean(s) ? TRUE : FALSE;
   }

   /**
    * Returns {@code true} if and only if the system property
    * named by the argument exists and is equal to the string
    * {@code "true"}. (Beginning with version 1.0.2 of the
    * Java<small><sup>TM</sup></small> platform, the test of
    * this string is case insensitive.) A system property is accessible
    * through {@code getProperty}, a method defined by the
    * {@code System} class.
    * <p>
    * If there is no property with the specified name, or if the specified
    * name is empty or null, then {@code false} is returned.
    *
    * 获取系统的属性,如果该属性存在且值不为空,并且值等于true(无视大小写),返回
    * true,否则返回false.
    *
    * @param   name   the system property name.
    * @return  the {@code boolean} value of the system property.
    * @throws  SecurityException for the same reasons as
    *          {@link System#getProperty(String) System.getProperty}
    * @see     java.lang.System#getProperty(java.lang.String)
    * @see     java.lang.System#getProperty(java.lang.String, java.lang.String)
    */
   public static boolean getBoolean(String name) {
      boolean result = false;
      try {
         result = parseBoolean(System.getProperty(name));
      } catch (IllegalArgumentException | NullPointerException e) {
      }
      return result;
   }

   /**
    * Compares two {@code boolean} values.
    * The value returned is identical(相同) to what would be returned by:
    * <pre>
    *    Boolean.valueOf(x).compareTo(Boolean.valueOf(y))
    * </pre>
    *
    * 比较两个boolean值,如果相同返回0,第一个为true返回1,否则返回-1
    *
    * @param  x the first {@code boolean} to compare
    * @param  y the second {@code boolean} to compare
    * @return the value {@code 0} if {@code x == y};
    *         a value less than {@code 0} if {@code !x && y}; and
    *         a value greater than {@code 0} if {@code x && !y}
    * @since 1.7
    */
   public static int compare(boolean x, boolean y) {
      return (x == y) ? 0 : (x ? 1 : -1);
   }

   /**
    * Returns the result of applying the logical AND operator to the
    * specified {@code boolean} operands.
    *
    * 返回逻辑的与运算(a和b都为true,为true,否则为false)
    *
    * @param a the first operand
    * @param b the second operand
    * @return the logical AND of {@code a} and {@code b}
    * @see java.util.function.BinaryOperator
    * @since 1.8
    */
   public static boolean logicalAnd(boolean a, boolean b) {
      return a && b;
   }

   /**
    * Returns the result of applying the logical OR operator to the
    * specified {@code boolean} operands.
    *
    * 返回逻辑的或运算(a和b存在true,为true,否则为false).
    *
    * @param a the first operand
    * @param b the second operand
    * @return the logical OR of {@code a} and {@code b}
    * @see java.util.function.BinaryOperator
    * @since 1.8
    */
   public static boolean logicalOr(boolean a, boolean b) {
      return a || b;
   }

   /**
    * Returns the result of applying the logical XOR operator to the
    * specified {@code boolean} operands.
    *
    * 返回逻辑的异或运算(a和b不相同为true,否则为false)
    *
    * @param a the first operand
    * @param b the second operand
    * @return  the logical XOR of {@code a} and {@code b}
    * @see java.util.function.BinaryOperator
    * @since 1.8
    */
   public static boolean logicalXor(boolean a, boolean b) {
      return a ^ b;
   }
}
```
