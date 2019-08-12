# Float

## IEEE754简述

IEEE754: `The IEEE Standard for Floating-Point Arithmetic (IEEE 754) is a technical standard for floating-point computation established in 1985 by the Institute of Electrical and Electronics Engineers (IEEE).`; 其中Java中的float则对应IEEE754中的单精度浮点数, double则对应IEEE754中的双精度浮点数.这里简述一下IEEE754中对浮点数的定义, 详细请查阅该文档或[wikipedia](https://en.wikipedia.org/wiki/IEEE_754).

IEEE754中32位浮点数的格式(也就是单精度浮点数,float)为: `1bit符号 + 8bit阶码 + 23bit尾数`. 共32位bit.

+ 1bit符号: 用1个bit位来存储正负值, 0代表正浮点数, 1表示负浮点数.
+ 8bit阶码: 用8个bit位来存储阶码, 这里用的是无符号值, 即最大值为255(2^8 - 1). 但是使用`偏移量(biased)`的方式来区分正负, 默认为最大值的一半127(255/2). 即实际的阶码值为: 阶码 - 偏移量(这里为127).
+ 23bit尾数: 用23个bit位来存储尾数, 即有效位数, 注意这里有一个隐藏的尾数1(默认的约定, 用于规范浮点数的表示, 详细看下面的范例), 即实际的尾数值应该为: 1 + 尾数.

值计算方式: (-1)^符号 x (1 + 尾数) x 2^(阶码 - 127).

如2进制的: -0.11, 是如何存储的呢?

+ 首先转换为标准的计算格式: -0.11 = (-1)^1 x (1 + 0.1) x 2^(-1).
+ 符号位肯定是1, 表明这个浮点数是负数. 1位bit表示为1.
+ 阶码位为实际阶码位 + 127 (因为存储的值要减去127进行偏移), 为 -1 + 127 = 126, 8位bit表示为: 01111110.
+ 尾数为1.1(注意刚开始的时候为0.11, 不符合隐藏尾数1的约定, 进行了进位操作以满足要求), 实际存储在尾数中的为0.1. 23位bit表示为: 10000...(22个0)
+ 最终存储在内存中数据为(用`,`区分不同的位含义): 1,01111110,10000...(22个0)

那10进制的: 100.25如何存储的呢?

+ 首先转换为二进制数为: 1100100.01
+ 转换为标准的计算格式: (-1)^0 x (1 + 0.10010001) x 2^6
+ 符号位为0, 表明这个浮点数是正数. 1位bit表示为0.
+ 阶码位为实际阶码位 + 127 (因为存储的值要减去127进行偏移), 为 6 + 127 = 133, 8位bit表示为: 10000101.
+ 尾数为1.10010001(注意刚开始的时候为1100100, 不符合隐藏尾数1的约定, 进行了进位操作以满足要求), 实际存储在尾数中的为0.10010001. 23位bit表示为: 10010001000...(15个0)
+ 最终存储在内存中数据为(用`,`区分不同的位含义): 0,10000101,10010001000...(15个0)

特殊的值:

+ 真值0: 阶码位和尾数位都为0, 根据符号位的区分可以分为正真值0和负真值0. 一般阶码的最小值-127(0-127)不用在范围内, 而是用来表示0这一特殊标示.
+ 无穷大: 阶码位全为1, 尾数位全为0, 则视为无穷大, 根据符号位的区分可以分为正无穷大和负无穷大. 所以一般阶码的最大值128(255-127)不用在范围内, 而是用于表示无穷大这个特殊标记.
+ 由于上面两个特殊位的占据, 阶码的范围在从`-(最大值/2 - 1)`到`最大值/2`之间.

范围和精度:

+ float和double的组成: Float, `1bit符号 +8bit阶码 + 23bit尾数`. Double, `1bit符号 +11bit阶码 + 52bit尾数`.
+ 由于尾数默认为1 + 尾数(尾数全是小数点之后的值), 主要是由阶码进行控制. 阶码最大值代表绝对值的最大的数,阶码的最小值则代表绝对值最小的数. float位: -2^126 - 2^127. double: -2^1022 - 2^1023(注意,这里只是粗略值).
+ 精度则是主要由尾数组成, float: 2^23 = 8388608(共7位), double: 2^52 = 4503599627370496(共16位), 所以float的精度为6-7位, double的精度为15-16位.

```java
package java.lang;

import sun.misc.FloatingDecimal;
import sun.misc.FloatConsts;
import sun.misc.DoubleConsts;

/**
 * The {@code Float} class wraps a value of primitive type
 * {@code float} in an object. An object of type
 * {@code Float} contains a single field whose type is
 * {@code float}.
 *
 * <p>In addition, this class provides several methods for converting a
 * {@code float} to a {@code String} and a
 * {@code String} to a {@code float}, as well as other
 * constants and methods useful when dealing with a
 * {@code float}.
 *
 * @author  Lee Boynton
 * @author  Arthur van Hoff
 * @author  Joseph D. Darcy
 * @since JDK1.0
 */
public final class Float extends Number implements Comparable<Float> {
	/**
	 * A constant holding the positive infinity of type
	 * {@code float}. It is equal to the value returned by
	 * {@code Float.intBitsToFloat(0x7f800000)}.
	 *
	 *
	 * 正无穷大, 大小等价于: Float.intBitsToFloat(0x7f800000)
	 * 0,11111111,00...(23个0): 符号为0, 阶码位全为1, 尾数全为0. 
	 * 表示正无穷大.
	 */
	public static final float POSITIVE_INFINITY = 1.0f / 0.0f;

	/**
	 * A constant holding the negative infinity of type
	 * {@code float}. It is equal to the value returned by
	 * {@code Float.intBitsToFloat(0xff800000)}.
	 *
	 * 负无穷大, 大小等价于: Float.intBitsToFloat(0xff800000)
	 * 1,11111111,00...(23个0): 符号为1, 阶码位全为1, 尾数全为0. 
	 * 表示负无穷大.
	 */
	public static final float NEGATIVE_INFINITY = -1.0f / 0.0f;

	/**
	 * A constant holding a Not-a-Number (NaN) value of type
	 * {@code float}.  It is equivalent to the value returned by
	 * {@code Float.intBitsToFloat(0x7fc00000)}.
	 *
	 * 非数字常量, 大小等价于: Float.intBitsToFloat(0x7fc00000)
	 * 0,11111111,10...(22个0): 符号为0, 阶码位全为1, 尾数全为1.1. 
	 * 表示Not-a-Number.
	 */
	public static final float NaN = 0.0f / 0.0f;

	/**
	 * A constant holding the largest positive finite value of type
	 * {@code float}, (2-2<sup>-23</sup>)&middot;2<sup>127</sup>.
	 * It is equal to the hexadecimal floating-point literal
	 * {@code 0x1.fffffeP+127f} and also equal to
	 * {@code Float.intBitsToFloat(0x7f7fffff)}.
	 *
	 * 最大值(绝对值最大值), 大小等价于: Float.intBitsToFloat(0x7f7fffff)
	 * 0,11111110,111111...(23个1): 符号位为0, 阶码为最大值(127), 
	 * 尾数全为1.
	 * 值为0x1.fffffeP(尾数只有23位) x 2^127. 相当于10进制中的:3.4028235 x 10^38.
	 */
	public static final float MAX_VALUE = 0x1.fffffeP+127f; // 3.4028235e+38f

	/**
	 * A constant holding the smallest positive normal value of type
	 * {@code float}, 2<sup>-126</sup>.  It is equal to the
	 * hexadecimal floating-point literal {@code 0x1.0p-126f} and also
	 * equal to {@code Float.intBitsToFloat(0x00800000)}.
	 *
	 * 最小值(绝对值相对最小值), 大小等价于: Float.intBitsToFloat(0x00800000)
	 * 0,00000001,0000000(23个0): 符号位为0, 阶码为最小值-126(1-127), 
	 * 尾数全为0.
	 * 值为0x1.0 x 2^-126. 相当于10进制中的:1.17549435 x 10^-38.
	 *
	 * @since 1.6
	 */
	public static final float MIN_NORMAL = 0x1.0p-126f; // 1.17549435E-38f

	/**
	 * A constant holding the smallest positive nonzero value of type
	 * {@code float}, 2<sup>-149</sup>. It is equal to the
	 * hexadecimal floating-point literal {@code 0x0.000002P-126f}
	 * and also equal to {@code Float.intBitsToFloat(0x1)}.
	 *
	 * 绝对最小值(绝对值绝对最小值(没有隐藏尾数1)), 大小等价于: Float.intBitsToFloat(0x1)
	 * 0,00000001,000...1(22个0): 符号位为0, 阶码为最小值-126(1-127),尾数为
	 * 0x000002(即最后一个小数为1). 值为0x0.000002 x 2^-126. 
	 * 相当于10进制中的:1.4 x 10^-45.
	 * 注意这里的格式不是标准的格式(没有隐藏1)。
	 *
	 */
	public static final float MIN_VALUE = 0x0.000002P-126f; // 1.4e-45f

	/**
	 * Maximum exponent a finite {@code float} variable may have.  It
	 * is equal to the value returned by {@code
	 * Math.getExponent(Float.MAX_VALUE)}.
	 *
	 * 最大的指数, 即阶码的最大值为127.(128用来表示无穷大)
	 *
	 * @since 1.6
	 */
	public static final int MAX_EXPONENT = 127;

	/**
	 * Minimum exponent a normalized {@code float} variable may have.
	 * It is equal to the value returned by {@code
	 * Math.getExponent(Float.MIN_NORMAL)}.
	 *
	 * 最小的指数, 即阶码的最小值为-126.(-127用来表示0)
	 *
	 * @since 1.6
	 */
	public static final int MIN_EXPONENT = -126;

	/**
	 * The number of bits used to represent a {@code float} value.
	 *
	 * 占据32bit位.
	 *
	 * @since 1.5
	 */
	public static final int SIZE = 32;

	/**
	 * The number of bytes used to represent a {@code float} value.
	 *
	 * 占据8byte字节.
	 *
	 * @since 1.8
	 */
	public static final int BYTES = SIZE / Byte.SIZE;

	/**
	 * The {@code Class} instance representing the primitive type
	 * {@code float}.
	 *
	 * 原始类型float在JVM中定义的Class对象
	 *
	 * @since JDK1.1
	 */
	@SuppressWarnings("unchecked")
	public static final Class<Float> TYPE = (Class<Float>) Class.getPrimitiveClass("float");

	/**
	 * Returns a string representation of the {@code float}
	 * argument. All characters mentioned below are ASCII characters.
	 *
	 * 返回字符串表示的float. 如果该值的绝对值大于0.001, 小于10^7, 那么就使用正常的10进制表达方式
	 * (如0.001, 1232.123). 如果不在这个范围内的话, 就使用科学计数发.(如0.0001: 1.0E-4). 特殊情况,
	 * 负数前面添加负号-, 零则用0.0来表示, 负零返回-0.0, 非数字返回NaN, 正无穷大返回Infinity, 
	 * 负无穷大返回-Infinity. 如果想要自定义String的表现形式 (如显示几位小数)可以使用 
	 * java.text.NumberFormat进行格式化输出.
	 *
	 * <ul>
	 * <li>If the argument is NaN, the result is the string
	 * "{@code NaN}".
	 * <li>Otherwise, the result is a string that represents the sign and
	 *     magnitude (absolute value) of the argument. If the sign is
	 *     negative, the first character of the result is
	 *     '{@code -}' ({@code '\u005Cu002D'}); if the sign is
	 *     positive, no sign character appears in the result. As for
	 *     the magnitude <i>m</i>:
	 * <ul>
	 * <li>If <i>m</i> is infinity, it is represented by the characters
	 *     {@code "Infinity"}; thus, positive infinity produces
	 *     the result {@code "Infinity"} and negative infinity
	 *     produces the result {@code "-Infinity"}.
	 * <li>If <i>m</i> is zero, it is represented by the characters
	 *     {@code "0.0"}; thus, negative zero produces the result
	 *     {@code "-0.0"} and positive zero produces the result
	 *     {@code "0.0"}.
	 * <li> If <i>m</i> is greater than or equal to 10<sup>-3</sup> but
	 *      less than 10<sup>7</sup>, then it is represented as the
	 *      integer part of <i>m</i>, in decimal form with no leading
	 *      zeroes, followed by '{@code .}'
	 *      ({@code '\u005Cu002E'}), followed by one or more
	 *      decimal digits representing the fractional part of
	 *      <i>m</i>.
	 * <li> If <i>m</i> is less than 10<sup>-3</sup> or greater than or
	 *      equal to 10<sup>7</sup>, then it is represented in
	 *      so-called "computerized scientific notation." Let <i>n</i>
	 *      be the unique integer such that 10<sup><i>n</i> </sup>&le;
	 *      <i>m</i> {@literal <} 10<sup><i>n</i>+1</sup>; then let <i>a</i>
	 *      be the mathematically exact quotient of <i>m</i> and
	 *      10<sup><i>n</i></sup> so that 1 &le; <i>a</i> {@literal <} 10.
	 *      The magnitude is then represented as the integer part of
	 *      <i>a</i>, as a single decimal digit, followed by
	 *      '{@code .}' ({@code '\u005Cu002E'}), followed by
	 *      decimal digits representing the fractional part of
	 *      <i>a</i>, followed by the letter '{@code E}'
	 *      ({@code '\u005Cu0045'}), followed by a representation
	 *      of <i>n</i> as a decimal integer, as produced by the
	 *      method {@link java.lang.Integer#toString(int)}.
	 *
	 * </ul>
	 * </ul>
	 * How many digits must be printed for the fractional part of
	 * <i>m</i> or <i>a</i>? There must be at least one digit
	 * to represent the fractional part, and beyond that as many, but
	 * only as many, more digits as are needed to uniquely distinguish
	 * the argument value from adjacent values of type
	 * {@code float}. That is, suppose that <i>x</i> is the
	 * exact mathematical value represented by the decimal
	 * representation produced by this method for a finite nonzero
	 * argument <i>f</i>. Then <i>f</i> must be the {@code float}
	 * value nearest to <i>x</i>; or, if two {@code float} values are
	 * equally close to <i>x</i>, then <i>f</i> must be one of
	 * them and the least significant bit of the significand of
	 * <i>f</i> must be {@code 0}.
	 *
	 * <p>To create localized string representations of a floating-point
	 * value, use subclasses of {@link java.text.NumberFormat}.
	 *
	 * @param   f   the float to be converted.
	 * @return a string representation of the argument.
	 */
	public static String toString(float f) {
		//调用FloatingDecimal.toJavaFormatString进行转换
		return FloatingDecimal.toJavaFormatString(f);
	}

	/**
	 * Returns a hexadecimal string representation of the
	 * {@code float} argument. All characters mentioned below are
	 * ASCII characters.
	 *
	 * 返回十六进制表示float的String类型.如(0x1.fffffeP127, -0x1.3p15). 特殊情况:
	 * 非数字返回NaN, 负数前面添加负号-, 如果无穷大返回Infinity,
	 * 负无穷大返回-Infinity, 0返回0x0.0p0, 负0返回-0x0.0p0. 如果尾数全为0, 则返回
	 * 的string中使用0x1.0p来表示(默认尾数写全使用大写P), 尾数后余部分存在0, 则可以
	 * 省略不补充0, 但是需要使用p(小写)进行表示.
	 *
	 * <ul>
	 * <li>If the argument is NaN, the result is the string
	 *     "{@code NaN}".
	 * <li>Otherwise, the result is a string that represents the sign and
	 * magnitude (absolute value) of the argument. If the sign is negative,
	 * the first character of the result is '{@code -}'
	 * ({@code '\u005Cu002D'}); if the sign is positive, no sign character
	 * appears in the result. As for the magnitude <i>m</i>:
	 *
	 * <ul>
	 * <li>If <i>m</i> is infinity, it is represented by the string
	 * {@code "Infinity"}; thus, positive infinity produces the
	 * result {@code "Infinity"} and negative infinity produces
	 * the result {@code "-Infinity"}.
	 *
	 * <li>If <i>m</i> is zero, it is represented by the string
	 * {@code "0x0.0p0"}; thus, negative zero produces the result
	 * {@code "-0x0.0p0"} and positive zero produces the result
	 * {@code "0x0.0p0"}.
	 *
	 * <li>If <i>m</i> is a {@code float} value with a
	 * normalized representation, substrings are used to represent the
	 * significand(尾数) and exponent fields(阶码).  The significand is
	 * represented by the characters {@code "0x1."}
	 * followed by a lowercase hexadecimal representation of the rest
	 * of the significand as a fraction.  Trailing zeros in the
	 * hexadecimal representation are removed unless all the digits
	 * are zero, in which case a single zero is used. Next, the
	 * exponent is represented by {@code "p"} followed
	 * by a decimal string of the unbiased exponent as if produced by
	 * a call to {@link Integer#toString(int) Integer.toString} on the
	 * exponent value.
	 *
	 * <li>If <i>m</i> is a {@code float} value with a subnormal
	 * representation, the significand is represented by the
	 * characters {@code "0x0."} followed by a
	 * hexadecimal representation of the rest of the significand as a
	 * fraction.  Trailing zeros in the hexadecimal representation are
	 * removed. Next, the exponent is represented by
	 * {@code "p-126"}.  Note that there must be at
	 * least one nonzero digit in a subnormal significand.
	 *
	 * </ul>
	 *
	 * </ul>
	 *
	 * <table border>
	 * <caption>Examples</caption>
	 * <tr><th>Floating-point Value</th><th>Hexadecimal String</th>
	 * <tr><td>{@code 1.0}</td> <td>{@code 0x1.0p0}</td>
	 * <tr><td>{@code -1.0}</td>        <td>{@code -0x1.0p0}</td>
	 * <tr><td>{@code 2.0}</td> <td>{@code 0x1.0p1}</td>
	 * <tr><td>{@code 3.0}</td> <td>{@code 0x1.8p1}</td>
	 * <tr><td>{@code 0.5}</td> <td>{@code 0x1.0p-1}</td>
	 * <tr><td>{@code 0.25}</td>        <td>{@code 0x1.0p-2}</td>
	 * <tr><td>{@code Float.MAX_VALUE}</td>
	 *     <td>{@code 0x1.fffffep127}</td>
	 * <tr><td>{@code Minimum Normal Value}</td>
	 *     <td>{@code 0x1.0p-126}</td>
	 * <tr><td>{@code Maximum Subnormal Value}</td>
	 *     <td>{@code 0x0.fffffep-126}</td>
	 * <tr><td>{@code Float.MIN_VALUE}</td>
	 *     <td>{@code 0x0.000002p-126}</td>
	 * </table>
	 * @param   f   the {@code float} to be converted.
	 * @return a hex string representation of the argument.
	 * @since 1.5
	 * @author Joseph D. Darcy
	 */
	public static String toHexString(float f) {
		if (Math.abs(f) < FloatConsts.MIN_NORMAL
			&&  f != 0.0f ) {// float subnormal
			// Adjust exponent to create subnormal double, then
			// replace subnormal double exponent with subnormal float
			// exponent
			String s = Double.toHexString(Math.scalb((double)f,
													 /* -1022+126 */
													 DoubleConsts.MIN_EXPONENT-
													 FloatConsts.MIN_EXPONENT));
			return s.replaceFirst("p-1022$", "p-126");
		}
		else // double string will be the same as float string
			return Double.toHexString(f);
	}

	/**
	 * Returns a {@code Float} object holding the
	 * {@code float} value represented by the argument string
	 * {@code s}.
	 *
	 * 返回String类型代表的float值(用于和toString之间的相互转换).
	 *
	 * <p>If {@code s} is {@code null}, then a
	 * {@code NullPointerException} is thrown.
	 * 
	 * 如果传递的String为null, 则会抛出一个NullPointerException.
	 *
	 * <p>Leading and trailing whitespace characters in {@code s}
	 * are ignored.  Whitespace is removed as if by the {@link
	 * String#trim} method; that is, both ASCII space and control
	 * characters are removed. The rest of {@code s} should
	 * constitute a <i>FloatValue</i> as described by the lexical
	 * syntax rules:
	 *
	 * String中的空格和control字符串都会被移除, 之后的字符串必须满足
	 * 如下的语法规则: 正常的十进制小数(如1.23f), 10进制的科学计数法
	 * (如1.23e78), 十六进制的浮点数(0x0.fffffep-126).
	 * 并且在转换的过程中存在精度丢失的问题,JVM会尽可能地靠近对应值.
	 * 如果转换的值大于或等于(MAX_VALUE + Math.ulp(MAX_VALUE)/2)则会
	 * 返回Infinity,或者其绝对值小于或等于MIN_VALUE/2, 则会返回0.
	 * 同样需要注意的是,直接调用本方法获取float和先转换为double,再转
	 * 换为float有可能存在问题, 不会等价, 这里需要注意.
	 *
	 * <blockquote>
	 * <dl>
	 * <dt><i>FloatValue:</i>
	 * <dd><i>Sign<sub>opt</sub></i> {@code NaN}
	 * <dd><i>Sign<sub>opt</sub></i> {@code Infinity}
	 * <dd><i>Sign<sub>opt</sub> FloatingPointLiteral</i>
	 * <dd><i>Sign<sub>opt</sub> HexFloatingPointLiteral</i>
	 * <dd><i>SignedInteger</i>
	 * </dl>
	 *
	 * <dl>
	 * <dt><i>HexFloatingPointLiteral</i>:
	 * <dd> <i>HexSignificand BinaryExponent FloatTypeSuffix<sub>opt</sub></i>
	 * </dl>
	 *
	 * <dl>
	 * <dt><i>HexSignificand:</i>
	 * <dd><i>HexNumeral</i>
	 * <dd><i>HexNumeral</i> {@code .}
	 * <dd>{@code 0x} <i>HexDigits<sub>opt</sub>
	 *     </i>{@code .}<i> HexDigits</i>
	 * <dd>{@code 0X}<i> HexDigits<sub>opt</sub>
	 *     </i>{@code .} <i>HexDigits</i>
	 * </dl>
	 *
	 * <dl>
	 * <dt><i>BinaryExponent:</i>
	 * <dd><i>BinaryExponentIndicator SignedInteger</i>
	 * </dl>
	 *
	 * <dl>
	 * <dt><i>BinaryExponentIndicator:</i>
	 * <dd>{@code p}
	 * <dd>{@code P}
	 * </dl>
	 *
	 * </blockquote>
	 *
	 * where <i>Sign</i>, <i>FloatingPointLiteral</i>,
	 * <i>HexNumeral</i>, <i>HexDigits</i>, <i>SignedInteger</i> and
	 * <i>FloatTypeSuffix</i> are as defined in the lexical structure
	 * sections of
	 * <cite>The Java&trade; Language Specification</cite>,
	 * except that underscores are not accepted between digits.
	 * If {@code s} does not have the form of
	 * a <i>FloatValue</i>, then a {@code NumberFormatException}
	 * is thrown. Otherwise, {@code s} is regarded as
	 * representing an exact decimal value in the usual
	 * "computerized scientific notation" or as an exact
	 * hexadecimal value; this exact numerical value is then
	 * conceptually converted to an "infinitely precise"
	 * binary value that is then rounded to type {@code float}
	 * by the usual round-to-nearest rule of IEEE 754 floating-point
	 * arithmetic, which includes preserving the sign of a zero
	 * value.
	 *
	 * Note that the round-to-nearest rule also implies overflow and
	 * underflow behaviour; if the exact value of {@code s} is large
	 * enough in magnitude (greater than or equal to ({@link
	 * #MAX_VALUE} + {@link Math#ulp(float) ulp(MAX_VALUE)}/2),
	 * rounding to {@code float} will result in an infinity and if the
	 * exact value of {@code s} is small enough in magnitude (less
	 * than or equal to {@link #MIN_VALUE}/2), rounding to float will
	 * result in a zero.
	 *
	 * Finally, after rounding a {@code Float} object representing
	 * this {@code float} value is returned.
	 *
	 * <p>To interpret localized string representations of a
	 * floating-point value, use subclasses of {@link
	 * java.text.NumberFormat}.
	 *
	 * <p>Note that trailing format specifiers, specifiers that
	 * determine the type of a floating-point literal
	 * ({@code 1.0f} is a {@code float} value;
	 * {@code 1.0d} is a {@code double} value), do
	 * <em>not</em> influence the results of this method.  In other
	 * words, the numerical value of the input string is converted
	 * directly to the target floating-point type.  In general, the
	 * two-step sequence of conversions, string to {@code double}
	 * followed by {@code double} to {@code float}, is
	 * <em>not</em> equivalent to converting a string directly to
	 * {@code float}.  For example, if first converted to an
	 * intermediate {@code double} and then to
	 * {@code float}, the string<br>
	 * {@code "1.00000017881393421514957253748434595763683319091796875001d"}<br>
	 * results in the {@code float} value
	 * {@code 1.0000002f}; if the string is converted directly to
	 * {@code float}, <code>1.000000<b>1</b>f</code> results.
	 *
	 * <p>To avoid calling this method on an invalid string and having
	 * a {@code NumberFormatException} be thrown, the documentation
	 * for {@link Double#valueOf Double.valueOf} lists a regular
	 * expression which can be used to screen the input.
	 *
	 * @param   s   the string to be parsed.
	 * @return  a {@code Float} object holding the value
	 *          represented by the {@code String} argument.
	 * @throws  NumberFormatException  if the string does not contain a
	 *          parsable number.
	 */
	public static Float valueOf(String s) throws NumberFormatException {
		return new Float(parseFloat(s));
	}

	/**
	 * Returns a {@code Float} instance representing the specified
	 * {@code float} value.
	 * If a new {@code Float} instance is not required, this method
	 * should generally be used in preference to the constructor
	 * {@link #Float(float)}, as this method is likely to yield
	 * significantly better space and time performance by caching
	 * frequently requested values.
	 *
	 * Float的构造函数, 推荐重复使用该对象的链接或者缓存, 而不是每次
	 * 需要的时候进行新建, 可以带来性能和时间上的优势(如果频繁调用).
	 *
	 * @param  f a float value.
	 * @return a {@code Float} instance representing {@code f}.
	 * @since  1.5
	 */
	public static Float valueOf(float f) {
		return new Float(f);
	}

	/**
	 * Returns a new {@code float} initialized to the value
	 * represented by the specified {@code String}, as performed
	 * by the {@code valueOf} method of class {@code Float}.
	 *
	 * 将一个String转换为float值.
	 *
	 * @param  s the string to be parsed.
	 * @return the {@code float} value represented by the string
	 *         argument.
	 * @throws NullPointerException  if the string is null
	 * @throws NumberFormatException if the string does not contain a
	 *               parsable {@code float}.
	 * @see    java.lang.Float#valueOf(String)
	 * @since 1.2
	 */
	public static float parseFloat(String s) throws NumberFormatException {
		return FloatingDecimal.parseFloat(s);
	}

	/**
	 * Returns {@code true} if the specified number is a
	 * Not-a-Number (NaN) value, {@code false} otherwise.
	 *
	 * 判断一个数是否是NaN值.
	 *
	 * @param   v   the value to be tested.
	 * @return  {@code true} if the argument is NaN;
	 *          {@code false} otherwise.
	 */
	public static boolean isNaN(float v) {
		return (v != v);
	}

	/**
	 * Returns {@code true} if the specified number is infinitely
	 * large in magnitude, {@code false} otherwise.
	 *
	 * 判断一个浮点数是否是无穷大.
	 *
	 * @param   v   the value to be tested.
	 * @return  {@code true} if the argument is positive infinity or
	 *          negative infinity; {@code false} otherwise.
	 */
	public static boolean isInfinite(float v) {
		return (v == POSITIVE_INFINITY) || (v == NEGATIVE_INFINITY);
	}


	/**
	 * Returns {@code true} if the argument is a finite floating-point
	 * value; returns {@code false} otherwise (for NaN and infinity
	 * arguments).
	 *
	 * 	判断一个浮点数是否是有穷大.
	 *
	 * @param f the {@code float} value to be tested
	 * @return {@code true} if the argument is a finite
	 * floating-point value, {@code false} otherwise.
	 * @since 1.8
	 */
	 public static boolean isFinite(float f) {
		return Math.abs(f) <= FloatConsts.MAX_VALUE;
	}

	/**
	 * The value of the Float.
	 *
	 * 封装的float值
	 *
	 * @serial
	 */
	private final float value;

	/**
	 * Constructs a newly allocated {@code Float} object that
	 * represents the primitive {@code float} argument.
	 *
	 * 构造函数
	 *
	 * @param   value   the value to be represented by the {@code Float}.
	 */
	public Float(float value) {
		this.value = value;
	}

	/**
	 * Constructs a newly allocated {@code Float} object that
	 * represents the argument converted to type {@code float}.
	 * 构造函数(double)
	 * @param   value   the value to be represented by the {@code Float}.
	 */
	public Float(double value) {
		this.value = (float)value;
	}

	/**
	 * Constructs a newly allocated {@code Float} object that
	 * represents the floating-point value of type {@code float}
	 * represented by the string. The string is converted to a
	 * {@code float} value as if by the {@code valueOf} method.
	 * 构造函数(float)
	 * @param      s   a string to be converted to a {@code Float}.
	 * @throws  NumberFormatException  if the string does not contain a
	 *               parsable number.
	 * @see        java.lang.Float#valueOf(java.lang.String)
	 */
	public Float(String s) throws NumberFormatException {
		value = parseFloat(s);
	}

	/**
	 * Returns {@code true} if this {@code Float} value is a
	 * Not-a-Number (NaN), {@code false} otherwise.
	 * 是否是NaN值.
	 * @return  {@code true} if the value represented by this object is
	 *          NaN; {@code false} otherwise.
	 */
	public boolean isNaN() {
		return isNaN(value);
	}

	/**
	 * Returns {@code true} if this {@code Float} value is
	 * infinitely large in magnitude, {@code false} otherwise.
	 * 是否是无穷大.
	 * @return  {@code true} if the value represented by this object is
	 *          positive infinity or negative infinity;
	 *          {@code false} otherwise.
	 */
	public boolean isInfinite() {
		return isInfinite(value);
	}

	/**
	 * Returns a string representation of this {@code Float} object.
	 * The primitive {@code float} value represented by this object
	 * is converted to a {@code String} exactly as if by the method
	 * {@code toString} of one argument.
	 * 转换为String对象.
	 * @return  a {@code String} representation of this object.
	 * @see java.lang.Float#toString(float)
	 */
	public String toString() {
		return Float.toString(value);
	}

	//下面为Number的接口实现.
	/**
	 * Returns the value of this {@code Float} as a {@code byte} after
	 * a narrowing primitive conversion.
	 *
	 * @return  the {@code float} value represented by this object
	 *          converted to type {@code byte}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public byte byteValue() {
		return (byte)value;
	}

	/**
	 * Returns the value of this {@code Float} as a {@code short}
	 * after a narrowing primitive conversion.
	 *
	 * @return  the {@code float} value represented by this object
	 *          converted to type {@code short}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 * @since JDK1.1
	 */
	public short shortValue() {
		return (short)value;
	}

	/**
	 * Returns the value of this {@code Float} as an {@code int} after
	 * a narrowing primitive conversion.
	 *
	 * @return  the {@code float} value represented by this object
	 *          converted to type {@code int}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public int intValue() {
		return (int)value;
	}

	/**
	 * Returns value of this {@code Float} as a {@code long} after a
	 * narrowing primitive conversion.
	 *
	 * @return  the {@code float} value represented by this object
	 *          converted to type {@code long}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public long longValue() {
		return (long)value;
	}

	/**
	 * Returns the {@code float} value of this {@code Float} object.
	 *
	 * @return the {@code float} value represented by this object
	 */
	public float floatValue() {
		return value;
	}

	/**
	 * Returns the value of this {@code Float} as a {@code double}
	 * after a widening primitive conversion.
	 *
	 * @return the {@code float} value represented by this
	 *         object converted to type {@code double}
	 * @jls 5.1.2 Widening Primitive Conversions
	 */
	public double doubleValue() {
		return (double)value;
	}

	/**
	 * Returns a hash code for this {@code Float} object. The
	 * result is the integer bit representation, exactly as produced
	 * by the method {@link #floatToIntBits(float)}, of the primitive
	 * {@code float} value represented by this {@code Float}
	 * object.
	 *
	 * 返回Float对象的哈希码.
	 * @return a hash code value for this object.
	 */
	@Override
	public int hashCode() {
		return Float.hashCode(value);
	}

	/**
	 * Returns a hash code for a {@code float} value; compatible with
	 * {@code Float.hashCode()}.
	 * 
	 * Float哈希码的计算方法, 转换位对应32bit位的int类型数值.
	 *
	 * @param value the value to hash
	 * @return a hash code value for a {@code float} value.
	 * @since 1.8
	 */
	public static int hashCode(float value) {
		return floatToIntBits(value);
	}

	/**
	 * Compares this object against the specified object.  The result
	 * is {@code true} if and only if the argument is not
	 * {@code null} and is a {@code Float} object that
	 * represents a {@code float} with the same value as the
	 * {@code float} represented by this object. For this
	 * purpose, two {@code float} values are considered to be the
	 * same if and only if the method {@link #floatToIntBits(float)}
	 * returns the identical {@code int} value when applied to
	 * each.
	 *
	 * 判断两个Float对象是否相同, 判断其内部封装的float值.
	 *
	 * <p>Note that in most cases, for two instances of class
	 * {@code Float}, {@code f1} and {@code f2}, the value
	 * of {@code f1.equals(f2)} is {@code true} if and only if
	 *
	 * <blockquote><pre>
	 *   f1.floatValue() == f2.floatValue()
	 * </pre></blockquote>
	 *
	 * <p>also has the value {@code true}. However, there are two exceptions:
	 * <ul>
	 * <li>If {@code f1} and {@code f2} both represent
	 *     {@code Float.NaN}, then the {@code equals} method returns
	 *     {@code true}, even though {@code Float.NaN==Float.NaN}
	 *     has the value {@code false}.
	 * <li>If {@code f1} represents {@code +0.0f} while
	 *     {@code f2} represents {@code -0.0f}, or vice
	 *     versa, the {@code equal} test has the value
	 *     {@code false}, even though {@code 0.0f==-0.0f}
	 *     has the value {@code true}.
	 * </ul>
	 *
	 * This definition allows hash tables to operate properly.
	 *
	 * @param obj the object to be compared
	 * @return  {@code true} if the objects are the same;
	 *          {@code false} otherwise.
	 * @see java.lang.Float#floatToIntBits(float)
	 */
	public boolean equals(Object obj) {
		return (obj instanceof Float)
			   && (floatToIntBits(((Float)obj).value) == floatToIntBits(value));
	}

	/**
	 * Returns a representation of the specified floating-point value
	 * according to the IEEE 754 floating-point "single format" bit
	 * layout.
	 *
	 * 将Float转换为对应32bit的int值. 注意这里进行了范围校验.如果超出范围
	 * 统一返回一个相同的NaN值.
	 *
	 * <p>Bit 31 (the bit that is selected by the mask
	 * {@code 0x80000000}) represents the sign of the floating-point
	 * number. 符号位
	 * Bits 30-23 (the bits that are selected by the mask
	 * {@code 0x7f800000}) represent the exponent. 阶码位
	 * Bits 22-0 (the bits that are selected by the mask
	 * {@code 0x007fffff}) represent the significand (sometimes called
	 * the mantissa) of the floating-point number.
	 * 尾数位
	 * <p>If the argument is positive infinity, the result is
	 * {@code 0x7f800000}.
	 *
	 * <p>If the argument is negative infinity, the result is
	 * {@code 0xff800000}.
	 *
	 * <p>If the argument is NaN, the result is {@code 0x7fc00000}.
	 *
	 * <p>In all cases, the result is an integer that, when given to the
	 * {@link #intBitsToFloat(int)} method, will produce a floating-point
	 * value the same as the argument to {@code floatToIntBits}
	 * (except all NaN values are collapsed to a single
	 * "canonical" NaN value).
	 *
	 * @param   value   a floating-point number.
	 * @return the bits that represent the floating-point number.
	 */
	public static int floatToIntBits(float value) {
		int result = floatToRawIntBits(value);
		// Check for NaN based on values of bit fields, maximum
		// exponent and nonzero significand. 
		//判断是否规范,或者超出范围.
		if ( ((result & FloatConsts.EXP_BIT_MASK) ==
			  FloatConsts.EXP_BIT_MASK) &&
			 (result & FloatConsts.SIGNIF_BIT_MASK) != 0)
			result = 0x7fc00000;
		return result;
	}

	/**
	 * Returns a representation of the specified floating-point value
	 * according to the IEEE 754 floating-point "single format" bit
	 * layout, preserving Not-a-Number (NaN) values.
	 *
	 * 本地方法, 将float中使用的32bit转换为对应bit位的原生int值.
	 *
	 * <p>Bit 31 (the bit that is selected by the mask
	 * {@code 0x80000000}) represents the sign of the floating-point
	 * number.
	 * Bits 30-23 (the bits that are selected by the mask
	 * {@code 0x7f800000}) represent the exponent.
	 * Bits 22-0 (the bits that are selected by the mask
	 * {@code 0x007fffff}) represent the significand (sometimes called
	 * the mantissa) of the floating-point number.
	 *
	 * <p>If the argument is positive infinity, the result is
	 * {@code 0x7f800000}.
	 *
	 * <p>If the argument is negative infinity, the result is
	 * {@code 0xff800000}.
	 *
	 * <p>If the argument is NaN, the result is the integer representing
	 * the actual NaN value.  Unlike the {@code floatToIntBits}
	 * method, {@code floatToRawIntBits} does not collapse all the
	 * bit patterns encoding a NaN to a single "canonical"
	 * NaN value.
	 *
	 * <p>In all cases, the result is an integer that, when given to the
	 * {@link #intBitsToFloat(int)} method, will produce a
	 * floating-point value the same as the argument to
	 * {@code floatToRawIntBits}.
	 *
	 * @param   value   a floating-point number.
	 * @return the bits that represent the floating-point number.
	 * @since 1.3
	 */
	public static native int floatToRawIntBits(float value);

	/**
	 * Returns the {@code float} value corresponding to a given
	 * bit representation.
	 * The argument is considered to be a representation of a
	 * floating-point value according to the IEEE 754 floating-point
	 * "single format" bit layout.
	 *
	 * 将int的32bit位转换为对应位置的float值. 需要进行范围校验.
	 *
	 * <p>If the argument is {@code 0x7f800000}, the result is positive
	 * infinity.
	 *
	 * <p>If the argument is {@code 0xff800000}, the result is negative
	 * infinity.
	 *
	 * <p>If the argument is any value in the range
	 * {@code 0x7f800001} through {@code 0x7fffffff} or in
	 * the range {@code 0xff800001} through
	 * {@code 0xffffffff}, the result is a NaN.  No IEEE 754
	 * floating-point operation provided by Java can distinguish
	 * between two NaN values of the same type with different bit
	 * patterns.  Distinct values of NaN are only distinguishable by
	 * use of the {@code Float.floatToRawIntBits} method.
	 * 不同的NaN值需要调用Float.floatToRawIntBits进行区分, 不能直接进行判断.
	 *
	 * <p>In all other cases, let <i>s</i>, <i>e</i>, and <i>m</i> be three
	 * values that can be computed from the argument:
	 *
	 * <blockquote><pre>{@code
	 * int s = ((bits >> 31) == 0) ? 1 : -1; 			//符号位
	 * int e = ((bits >> 23) & 0xff); 					//阶码位
	 * int m = (e == 0) ?
	 *                 (bits & 0x7fffff) << 1 :
	 *                 (bits & 0x7fffff) | 0x800000; 	//尾数位
	 * }</pre></blockquote>
	 *
	 * Then the floating-point result equals the value of the mathematical
	 * expression <i>s</i>&middot;<i>m</i>&middot;2<sup><i>e</i>-150</sup>.
	 *
	 * <p>Note that this method may not be able to return a
	 * {@code float} NaN with exactly same bit pattern as the
	 * {@code int} argument.  IEEE 754 distinguishes between two
	 * kinds of NaNs, quiet NaNs and <i>signaling NaNs</i>.  The
	 * differences between the two kinds of NaN are generally not
	 * visible in Java.  Arithmetic operations on signaling NaNs turn
	 * them into quiet NaNs with a different, but often similar, bit
	 * pattern.  However, on some processors merely copying a
	 * signaling NaN also performs that conversion.  In particular,
	 * copying a signaling NaN to return it to the calling method may
	 * perform this conversion.  So {@code intBitsToFloat} may
	 * not be able to return a {@code float} with a signaling NaN
	 * bit pattern.  Consequently, for some {@code int} values,
	 * {@code floatToRawIntBits(intBitsToFloat(start))} may
	 * <i>not</i> equal {@code start}.  Moreover, which
	 * particular bit patterns represent signaling NaNs is platform
	 * dependent; although all NaN bit patterns, quiet or signaling,
	 * must be in the NaN range identified above.
	 *
	 * 这个方法可能返回的NaN值不是对应传递的int的bit位所对应的. 对于
	 * IEEE754中定义的quiet NaNs和signaling NaNs, 在Java中是不可区分的.
	 * 并且signaling NaNs取决于平台实现(兼容性存在问题), 并且需要在定义
	 * 的范围内(0x7f800001-0x7fffffff,0xff800001-0xffffffff).
	 *
	 * @param   bits   an integer.
	 * @return  the {@code float} floating-point value with the same bit
	 *          pattern.
	 */
	public static native float intBitsToFloat(int bits);

	/**
	 * Compares two {@code Float} objects numerically.  There are
	 * two ways in which comparisons performed by this method differ
	 * from those performed by the Java language numerical comparison
	 * operators ({@code <, <=, ==, >=, >}) when
	 * applied to primitive {@code float} values:
	 *
	 * Comparable接口的实现函数.等0大1小-1.
	 *
	 * <ul><li>
	 *          {@code Float.NaN} is considered by this method to
	 *          be equal to itself and greater than all other
	 *          {@code float} values
	 *          (including {@code Float.POSITIVE_INFINITY}).
	 * <li>
	 *          {@code 0.0f} is considered by this method to be greater
	 *          than {@code -0.0f}.
	 * </ul>
	 *
	 * This ensures that the <i>natural ordering</i> of {@code Float}
	 * objects imposed by this method is <i>consistent with equals</i>.
	 *
	 * @param   anotherFloat   the {@code Float} to be compared.
	 * @return  the value {@code 0} if {@code anotherFloat} is
	 *          numerically equal to this {@code Float}; a value
	 *          less than {@code 0} if this {@code Float}
	 *          is numerically less than {@code anotherFloat};
	 *          and a value greater than {@code 0} if this
	 *          {@code Float} is numerically greater than
	 *          {@code anotherFloat}.
	 *
	 * @since   1.2
	 * @see Comparable#compareTo(Object)
	 */
	public int compareTo(Float anotherFloat) {
		return Float.compare(value, anotherFloat.value);
	}

	/**
	 * Compares the two specified {@code float} values. The sign
	 * of the integer value returned is the same as that of the
	 * integer that would be returned by the call:
	 * <pre>
	 *    new Float(f1).compareTo(new Float(f2))
	 * </pre>
	 *
	 * 静态工具方法来实现比较两个float值.等0大1小-1.
	 *
	 *
	 * @param   f1        the first {@code float} to compare.
	 * @param   f2        the second {@code float} to compare.
	 * @return  the value {@code 0} if {@code f1} is
	 *          numerically equal to {@code f2}; a value less than
	 *          {@code 0} if {@code f1} is numerically less than
	 *          {@code f2}; and a value greater than {@code 0}
	 *          if {@code f1} is numerically greater than
	 *          {@code f2}.
	 * @since 1.4
	 */
	public static int compare(float f1, float f2) {
		//如果f1,f2都不是NaN的话, 直接通过<>进行比较.
		if (f1 < f2)
			return -1;           // Neither val is NaN, thisVal is smaller
		if (f1 > f2)
			return 1;            // Neither val is NaN, thisVal is larger
		
		//这里使用floatToIntBits转换为对应位的int值, 而不是使用floatToRawIntBits.
		//因为floatToIntBits会对NaN进行过滤, 统一返回一个固定值.
		// Cannot use floatToRawIntBits because of possibility of NaNs.
		int thisBits    = Float.floatToIntBits(f1);
		int anotherBits = Float.floatToIntBits(f2);

		return (thisBits == anotherBits ?  0 : // Values are equal
				(thisBits < anotherBits ? -1 : // (-0.0, 0.0) or (!NaN, NaN)
				 1));                          // (0.0, -0.0) or (NaN, !NaN)
	}

	/**
	 * Adds two {@code float} values together as per the + operator.
	 * 求和
	 * @param a the first operand
	 * @param b the second operand
	 * @return the sum of {@code a} and {@code b}
	 * @jls 4.2.4 Floating-Point Operations
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static float sum(float a, float b) {
		return a + b;
	}

	/**
	 * Returns the greater of two {@code float} values
	 * as if by calling {@link Math#max(float, float) Math.max}.
	 * 求较大值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the greater of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static float max(float a, float b) {
		return Math.max(a, b);
	}

	/**
	 * Returns the smaller of two {@code float} values
	 * as if by calling {@link Math#min(float, float) Math.min}.
	 * 求较小值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the smaller of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static float min(float a, float b) {
		return Math.min(a, b);
	}

	/** use serialVersionUID from JDK 1.0.2 for interoperability */
	private static final long serialVersionUID = -2671257302660747028L;
}
```
