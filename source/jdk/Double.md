# Double

## IEEE754简述
IEEE754: `The IEEE Standard for Floating-Point Arithmetic (IEEE 754) is a technical standard for floating-point computation established in 1985 by the Institute of Electrical and Electronics Engineers (IEEE).`; 其中Java中的float则对应IEEE754中的单精度浮点数, double则对应IEEE754中的双精度浮点数.这里简述一下IEEE754中对浮点数的定义, 详细请查阅该文档或[wikipedia](https://en.wikipedia.org/wiki/IEEE_754).

IEEE754中32位浮点数的格式(也就是单精度浮点数,float)为: `1bit符号 + 8bit阶码 + 23bit尾数`. 共32位bit. 

+ 1bit符号: 用1个bit位来存储正负值, 0代表正浮点数, 1表示负浮点数. 
+ 8bit阶码: 用8个bit位来存储阶码, 这里用的是无符号值, 即最大值为255(2^8 - 1). 但是使用`偏移量(biased)`的方式来区分正负, 默认为最大值的一半127(255/2)(double为1023). 即实际的阶码值为: 阶码 - 偏移量(这里为127).
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
+ 阶码位为实际阶码位 + 127 (因为存储的值要减去127进行偏移, double的偏移量为1023), 为 6 + 127 = 133, 8位bit表示为: 10000101.
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

/**
 * The {@code Double} class wraps a value of the primitive type
 * {@code double} in an object. An object of type
 * {@code Double} contains a single field whose type is
 * {@code double}.
 *
 * <p>In addition, this class provides several methods for converting a
 * {@code double} to a {@code String} and a
 * {@code String} to a {@code double}, as well as other
 * constants and methods useful when dealing with a
 * {@code double}.
 *
 * @author  Lee Boynton
 * @author  Arthur van Hoff
 * @author  Joseph D. Darcy
 * @since JDK1.0
 */
public final class Double extends Number implements Comparable<Double> {
	/**
	 *  A constant holding the positive infinity of type
	 * {@code double}. It is equal to the value returned by
	 * {@code Double.longBitsToDouble(0x7ff0000000000000L)}.
	 *
	 *
	 * 正无穷大, 大小等价于: Double.longBitsToDouble(0x7ff0000000000000L)
	 * 0,11111111111,00...(53个0): 符号位为0, 阶码位全为1(1024, 2047 - 1023), 尾数全为0. 
	 * 表示正无穷大.
	 */
	public static final double POSITIVE_INFINITY = 1.0 / 0.0;

	/**
	 * A constant holding the negative infinity of type
	 * {@code double}. It is equal to the value returned by
	 * {@code Double.longBitsToDouble(0xfff0000000000000L)}.
	 *
	 * 负无穷大, 大小等价于: Double.longBitsToDouble(0xfff0000000000000L)
	 * 1,11111111111,00...(53个0): 符号位为1, 阶码位全为1(1024, 2047 - 1023), 尾数全为0. 
	 * 表示负无穷大.
	 */
	public static final double NEGATIVE_INFINITY = -1.0 / 0.0;

	/**
	 * A constant holding a Not-a-Number (NaN) value of type
	 * {@code double}. It is equivalent to the value returned by
	 * {@code Double.longBitsToDouble(0x7ff8000000000000L)}.
	 *
	 * 非数字常量, 大小等价于: Double.longBitsToDouble(0x7ff8000000000000L)
	 * 0,11111111111,10...(52个0): 符号为0, 阶码位全为1, 尾数全为1.1. 
	 * 表示Not-a-Number.
	 */
	public static final double NaN = 0.0d / 0.0;

	/**
	 * A constant holding the largest positive finite value of type
	 * {@code double},
	 * (2-2<sup>-52</sup>)&middot;2<sup>1023</sup>.  It is equal to
	 * the hexadecimal floating-point literal
	 * {@code 0x1.fffffffffffffP+1023} and also equal to
	 * {@code Double.longBitsToDouble(0x7fefffffffffffffL)}.*
	 *
	 * 最大值(绝对值最大值), 大小等价于: Double.longBitsToDouble(0x7fefffffffffffffL)
	 * 0,11111111110,111111...(23个1): 符号位为0, 阶码为最大值(1023), 
	 * 尾数全为1. 相当于10进制中的:1.7976931348623157 x 10^308.
	 */
	public static final double MAX_VALUE = 0x1.fffffffffffffP+1023; // 1.7976931348623157e+308

	/**
	 * A constant holding the smallest positive normal value of type
	 * {@code double}, 2<sup>-1022</sup>.  It is equal to the
	 * hexadecimal floating-point literal {@code 0x1.0p-1022} and also
	 * equal to {@code Double.longBitsToDouble(0x0010000000000000L)}.
	 *
	 * 最小值(绝对值最小值), 大小等价于: Double.longBitsToDouble(0x0010000000000000L)
	 * 0,00000000001,0000000(53个0): 符号位为0, 阶码为最小值-1022(1-1023), 
	 * 尾数全为0.
	 * 值为0x1.0 x 2^-126. 相当于10进制中的:2.2250738585072014 x 10^-308.
	 *
	 * @since 1.6
	 */
	public static final double MIN_NORMAL = 0x1.0p-1022; // 2.2250738585072014E-308

	/**
	 * A constant holding the smallest positive nonzero value of type
	 * {@code double}, 2<sup>-1074</sup>. It is equal to the
	 * hexadecimal floating-point literal
	 * {@code 0x0.0000000000001P-1022} and also equal to
	 * {@code Double.longBitsToDouble(0x1L)}.
	 *
	 * 绝对最小值(绝对值相对最小值), 大小等价于: Double.longBitsToDouble(0x1L)
	 * 0,00000000001,000...1(22个0): 符号位为0, 阶码为最小值-1022(1-1023),尾数为
	 * 0x0000000000001(即最后一个小数为1). 值为0x0.0000000000001 x 2^-1022. 
	 * 相当于10进制中的:4.9 x 10^-324.
	 * 注意这里的格式不是标准的格式(没有隐藏1)。
	 */
	public static final double MIN_VALUE = 0x0.0000000000001P-1022; // 4.9e-324

	/**
	 * Maximum exponent a finite {@code double} variable may have.
	 * It is equal to the value returned by
	 * {@code Math.getExponent(Double.MAX_VALUE)}.
	 *
	 * 最大指数为1023(2^11 - 1 - 1023 - 1), 1024用来代表无穷大
	 *
	 * @since 1.6
	 */
	public static final int MAX_EXPONENT = 1023;

	/**
	 * Minimum exponent a normalized {@code double} variable may
	 * have.  It is equal to the value returned by
	 * {@code Math.getExponent(Double.MIN_NORMAL)}.
	 *
	 * 最小的指数, 即阶码的最小值为-1022.(-1023用来表示0)
	 * 
	 * @since 1.6
	 */
	public static final int MIN_EXPONENT = -1022;

	/**
	 * The number of bits used to represent a {@code double} value.
	 *
	 * 占据的bit位
	 *
	 * @since 1.5
	 */
	public static final int SIZE = 64;

	/**
	 * The number of bytes used to represent a {@code double} value.
	 *
	 * 占据的byte位
	 *
	 * @since 1.8
	 */
	public static final int BYTES = SIZE / Byte.SIZE;

	/**
	 * The {@code Class} instance representing the primitive type
	 * {@code double}.
	 *
	 * 原始类型double在JVM中定义的Class对象
	 * 
	 * @since JDK1.1
	 */
	@SuppressWarnings("unchecked")
	public static final Class<Double>   TYPE = (Class<Double>) Class.getPrimitiveClass("double");

	/**
	 * Returns a string representation of the {@code double}
	 * argument. All characters mentioned below are ASCII characters.
	 *
	 * 返回字符串表示的double. 如果该值的绝对值大于0.001, 小于10^7, 那么就使用正常的10进制表达方式
	 * (如0.001, 1232.123). 如果不在这个范围内的话, 就使用科学计数发.(如0.0001: 1.0E-4). 特殊情况,
	 * 负数前面添加负号-, 零则用0.0来表示, 负零返回-0.0, 非数字返回NaN, 正无穷大返回Infinity, 
	 * 负无穷大返回-Infinity. 如果想要自定义String的表现形式 (如显示几位小数)可以使用 
	 * java.text.NumberFormat进行格式化输出.
	 *
	 * <ul>
	 * <li>If the argument is NaN, the result is the string
	 *     "{@code NaN}".
	 * <li>Otherwise, the result is a string that represents the sign and
	 * magnitude (absolute value) of the argument. If the sign is negative,
	 * the first character of the result is '{@code -}'
	 * ({@code '\u005Cu002D'}); if the sign is positive, no sign character
	 * appears in the result. As for the magnitude <i>m</i>:
	 * <ul>
	 * <li>If <i>m</i> is infinity, it is represented by the characters
	 * {@code "Infinity"}; thus, positive infinity produces the result
	 * {@code "Infinity"} and negative infinity produces the result
	 * {@code "-Infinity"}.
	 *
	 * <li>If <i>m</i> is zero, it is represented by the characters
	 * {@code "0.0"}; thus, negative zero produces the result
	 * {@code "-0.0"} and positive zero produces the result
	 * {@code "0.0"}.
	 *
	 * <li>If <i>m</i> is greater than or equal to 10<sup>-3</sup> but less
	 * than 10<sup>7</sup>, then it is represented as the integer part of
	 * <i>m</i>, in decimal form with no leading zeroes, followed by
	 * '{@code .}' ({@code '\u005Cu002E'}), followed by one or
	 * more decimal digits representing the fractional part of <i>m</i>.
	 *
	 * <li>If <i>m</i> is less than 10<sup>-3</sup> or greater than or
	 * equal to 10<sup>7</sup>, then it is represented in so-called
	 * "computerized scientific notation." Let <i>n</i> be the unique
	 * integer such that 10<sup><i>n</i></sup> &le; <i>m</i> {@literal <}
	 * 10<sup><i>n</i>+1</sup>; then let <i>a</i> be the
	 * mathematically exact quotient of <i>m</i> and
	 * 10<sup><i>n</i></sup> so that 1 &le; <i>a</i> {@literal <} 10. The
	 * magnitude is then represented as the integer part of <i>a</i>,
	 * as a single decimal digit, followed by '{@code .}'
	 * ({@code '\u005Cu002E'}), followed by decimal digits
	 * representing the fractional part of <i>a</i>, followed by the
	 * letter '{@code E}' ({@code '\u005Cu0045'}), followed
	 * by a representation of <i>n</i> as a decimal integer, as
	 * produced by the method {@link Integer#toString(int)}.
	 * </ul>
	 * </ul>
	 * How many digits must be printed for the fractional part of
	 * <i>m</i> or <i>a</i>? There must be at least one digit to represent
	 * the fractional part, and beyond that as many, but only as many, more
	 * digits as are needed to uniquely distinguish the argument value from
	 * adjacent values of type {@code double}. That is, suppose that
	 * <i>x</i> is the exact mathematical value represented by the decimal
	 * representation produced by this method for a finite nonzero argument
	 * <i>d</i>. Then <i>d</i> must be the {@code double} value nearest
	 * to <i>x</i>; or if two {@code double} values are equally close
	 * to <i>x</i>, then <i>d</i> must be one of them and the least
	 * significant bit of the significand of <i>d</i> must be {@code 0}.
	 *
	 * <p>To create localized string representations of a floating-point
	 * value, use subclasses of {@link java.text.NumberFormat}.
	 *
	 * @param   d   the {@code double} to be converted.
	 * @return a string representation of the argument.
	 */
	public static String toString(double d) {
		return FloatingDecimal.toJavaFormatString(d);
	}
	
	/**
	 * Returns a hexadecimal string representation of the
	 * {@code double} argument. All characters mentioned below
	 * are ASCII characters.
	 *
	 * 返回十六进制表示double的String类型.如(0x1.fffffffffffffP+1023). 特殊情况:
	 * 非数字返回NaN, 负数前面添加负号-, 如果无穷大返回Infinity,
	 * 负无穷大返回-Infinity, 0返回0x0.0p0, 负0返回-0x0.0p0. 如果尾数全为0, 则返回
	 * 的string中使用0x1.0p来表示(默认尾数写全使用大写P), 尾数后余部分存在0, 则可以
	 * 省略不补充0, 但是需要使用p(小写)进行表示. 如果是次正规的数据(以0x0.开始的)
	 * 那么尾部的0省略,设置阶码为p-1022,然后进行计算显示.如(0x0.0000000000001p-1022);
	 *
	 * <ul>
	 * <li>If the argument is NaN, the result is the string
	 *     "{@code NaN}".
	 * <li>Otherwise, the result is a string that represents the sign
	 * and magnitude of the argument. If the sign is negative, the
	 * first character of the result is '{@code -}'
	 * ({@code '\u005Cu002D'}); if the sign is positive, no sign
	 * character appears in the result. As for the magnitude <i>m</i>:
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
	 * <li>If <i>m</i> is a {@code double} value with a
	 * normalized representation, substrings are used to represent the
	 * significand and exponent fields.  The significand is
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
	 * <li>If <i>m</i> is a {@code double} value with a subnormal
	 * representation, the significand is represented by the
	 * characters {@code "0x0."} followed by a
	 * hexadecimal representation of the rest of the significand as a
	 * fraction.  Trailing zeros in the hexadecimal representation are
	 * removed. Next, the exponent is represented by
	 * {@code "p-1022"}.  Note that there must be at
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
	 * <tr><td>{@code Double.MAX_VALUE}</td>
	 *     <td>{@code 0x1.fffffffffffffp1023}</td>
	 * <tr><td>{@code Minimum Normal Value}</td>
	 *     <td>{@code 0x1.0p-1022}</td>
	 * <tr><td>{@code Maximum Subnormal Value}</td>
	 *     <td>{@code 0x0.fffffffffffffp-1022}</td>
	 * <tr><td>{@code Double.MIN_VALUE}</td>
	 *     <td>{@code 0x0.0000000000001p-1022}</td>
	 * </table>
	 * @param   d   the {@code double} to be converted.
	 * @return a hex string representation of the argument.
	 * @since 1.5
	 * @author Joseph D. Darcy
	 */
	public static String toHexString(double d) {
		/*
		 * Modeled after the "a" conversion specifier in C99, section
		 * 7.19.6.1; however, the output of this method is more
		 * tightly specified.
		 */
		if (!isFinite(d) )
			// For infinity and NaN, use the decimal output.
			return Double.toString(d);
		else {
			// Initialized to maximum size of output.
			StringBuilder answer = new StringBuilder(24);
			//获取浮点数带符号的第一位数(通过Math.copySign来获取)来判断正负
			if (Math.copySign(1.0, d) == -1.0)    // value is negative,
				answer.append("-");                  // so append sign info
			//添加16进制前缀
			answer.append("0x");
			//取绝对值
			d = Math.abs(d);

			if(d == 0.0) {
				answer.append("0.0p0");
			} else {
				//判断是否小于MIN_NORMAL(即尾数全为0,阶码为最小值)
				//如果小于这个值,表示肯定是不标准的表现形式(即不是以0x1.开头)
				boolean subnormal = (d < DoubleConsts.MIN_NORMAL);

				// Isolate significand(尾数) bits and OR in a high-order bit
				// so that the string representation has a known
				// length.
				//获取尾数的有效位数, DoubleConsts.SIGNIF_BIT_MASK为
				//1,11111111111,111111111111111111111111111111111111111111111111111
				long signifBits = (Double.doubleToLongBits(d)
								   & DoubleConsts.SIGNIF_BIT_MASK) |
					0x1000000000000000L;
				
				//正规的使用0x1.开头,不正规使用0x0.开头.
				// Subnormal values have a 0 implicit bit; normal
				// values have a 1 implicit bit.
				answer.append(subnormal ? "0." : "1.");

				// Isolate the low-order 13 digits of the hex
				// representation.  If all the digits are zero,
				// replace with a single 0; otherwise, remove all
				// trailing zeros.
				//获取尾数(后52bit位),并移除其尾部的0
				String signif = Long.toHexString(signifBits).substring(3,16);
				answer.append(signif.equals("0000000000000") ? // 13 zeros
							  "0":
							  signif.replaceFirst("0{1,12}$", "")); //移除尾部的0
				//添加p
				answer.append('p');
				// If the value is subnormal, use the E_min exponent
				// value for double; otherwise, extract and report d's
				// exponent (the representation of a subnormal uses
				// E_min -1).
				//如果是不正规的浮点数,使用-1022(默认的值),否则计算其阶码位(Math.getExponent)
				answer.append(subnormal ?
							  DoubleConsts.MIN_EXPONENT:
							  Math.getExponent(d));
			}
			return answer.toString();
		}
	}

	/**
	 * Returns a {@code Double} object holding the
	 * {@code double} value represented by the argument string
	 * {@code s}.
	 *
	 * 返回String类型代表的double值(用于和toString之间的相互转换).
	 *
	 * <p>If {@code s} is {@code null}, then a
	 * {@code NullPointerException} is thrown.
	 *
	 * <p>Leading and trailing whitespace characters in {@code s}
	 * are ignored.  Whitespace is removed as if by the {@link
	 * String#trim} method; that is, both ASCII space and control
	 * characters are removed. The rest of {@code s} should
	 * constitute a <i>FloatValue</i> as described by the lexical
	 * syntax rules:
	 *
	 * String中的空格和control字符串都会被移除, 之后的字符串必须满足
	 * 如下的语法规则: 正常的十进制小数(如1.23d), 10进制的科学计数法
	 * (如1.23e78), 十六进制的浮点数(0x1.0p-1022).
	 * 并且在转换的过程中存在精度丢失的问题,JVM会尽可能地靠近对应值.
	 * 如果转换的值大于或等于(MAX_VALUE + Math.ulp(MAX_VALUE)/2)则会
	 * 返回Infinity,或者其绝对值小于或等于MIN_VALUE/2, 则会返回0.
	 * 同样需要注意的是,直接调用本方法获取float和先转换为double,再转
	 * 换为float有可能存在问题, 不会等价, 这里需要注意. 
	 * 为了可以正确进行Double的转换,这里提供一个验证的方式:
	 *
		final String Digits     = "(\\p{Digit}+)";  //匹配各种数字
		final String HexDigits  = "(\\p{XDigit}+)"; //匹配各类十六进制数
		//类似的匹配POSIX字符类还有:
		//1     \p{Lower}	小写字母字符:[a-z]。
		//2	    \p{Upper}	大写字母字符:[A-Z]。
		//3	    \p{ASCII}	所有ASCII:[\x00-\x7F]。
		//4	    \p{Alpha}	字母字符:[\p{Lower}\p{Upper}]。
		//5	    \p{Digit}	十进制数字:[0-9]。
		//6	    \p{Alnum}	字母数字字符:[\p{Alpha}\p{Digit}]。
		//7	    \p{Punct}	标点符号:!”#$%&’()*+,-./:;<=>?@[]^_>{Ι}< 其中一个。
		//8	    \p{Graph}	一个可视的字符: [\p{Alnum}\p{Punct}]。
		//9	    \p{Print}	可打印字符:[\p{Graph}\x20]。
		//10	\p{Blank}	空格或制表符:[ \t]。
		//11	\p{XDigit}	十六进制数字:[0-9a-fA-F]。
		//12	\p{Space}	空白字符:[ \t\n\x0B\f\r]

		// an exponent is 'e' or 'E' followed by an optionally
		// signed decimal integer.
		final String Exp        = "[eE][+-]?"+Digits;
		final String fpRegex    =
			//ASCII 0 - 32大部分为空格(空字符,换行,回车等)开头允许空格
			("[\\x00-\\x20]*"+  // Optional leading "whitespace"
			//可能存在+/-号, 字符串可能为NaN或者Infinity.
			"[+-]?(NaN|Infinity|" +
				//如果是10进制常有的有128.2f, 22d, 10E7, 2.2e-2, .32E-2, .02E-3
				"((("+Digits+"(\\.)?("+Digits+"?)("+Exp+")?)|(\\.("+Digits+")("+Exp+")?)|"+
					//如果是十六进制的话, 常用的有:0x1.0p-1022, .2aP23
					"(((0[xX]" + HexDigits + "(\\.)?)|(0[xX]" + HexDigits + "?(\\.)" + HexDigits + "))" +
					"[pP][+-]?" + Digits + "))" +
					//尾部可以以f,F,d,D结尾
					"[fFdD]?))" +
			"[\\x00-\\x20]*");// Optional trailing "whitespace"

		if (Pattern.matches(fpRegex, myString)) {
			Double.valueOf(myString); // Will not throw NumberFormatException
		}
		else {
		 // Perform suitable alternative action
		}
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
	 * binary value that is then rounded to type {@code double}
	 * by the usual round-to-nearest rule of IEEE 754 floating-point
	 * arithmetic, which includes preserving the sign of a zero
	 * value.
	 *
	 * Note that the round-to-nearest rule also implies overflow and
	 * underflow behaviour; if the exact value of {@code s} is large
	 * enough in magnitude (greater than or equal to ({@link
	 * #MAX_VALUE} + {@link Math#ulp(double) ulp(MAX_VALUE)}/2),
	 * rounding to {@code double} will result in an infinity and if the
	 * exact value of {@code s} is small enough in magnitude (less
	 * than or equal to {@link #MIN_VALUE}/2), rounding to float will
	 * result in a zero.
	 *
	 * Finally, after rounding a {@code Double} object representing
	 * this {@code double} value is returned.
	 *
	 * <p> To interpret localized string representations of a
	 * floating-point value, use subclasses of {@link
	 * java.text.NumberFormat}.
	 *
	 * <p>Note that trailing format specifiers, specifiers that
	 * determine the type of a floating-point literal
	 * ({@code 1.0f} is a {@code float} value;
	 * {@code 1.0d} is a {@code double} value), do
	 * <em>not</em> influence the results of this method.  In other
	 * words, the numerical value of the input string is converted
	 * directly to the target floating-point type.  The two-step
	 * sequence of conversions, string to {@code float} followed
	 * by {@code float} to {@code double}, is <em>not</em>
	 * equivalent to converting a string directly to
	 * {@code double}. For example, the {@code float}
	 * literal {@code 0.1f} is equal to the {@code double}
	 * value {@code 0.10000000149011612}; the {@code float}
	 * literal {@code 0.1f} represents a different numerical
	 * value than the {@code double} literal
	 * {@code 0.1}. (The numerical value 0.1 cannot be exactly
	 * represented in a binary floating-point number.)
	 *
	 * <p>To avoid calling this method on an invalid string and having
	 * a {@code NumberFormatException} be thrown, the regular
	 * expression below can be used to screen the input string:
	 *
	 * <pre>{@code
	 *  final String Digits     = "(\\p{Digit}+)";
	 *  final String HexDigits  = "(\\p{XDigit}+)";
	 *  // an exponent is 'e' or 'E' followed by an optionally
	 *  // signed decimal integer.
	 *  final String Exp        = "[eE][+-]?"+Digits;
	 *  final String fpRegex    =
	 *      ("[\\x00-\\x20]*"+  // Optional leading "whitespace"
	 *       "[+-]?(" + // Optional sign character
	 *       "NaN|" +           // "NaN" string
	 *       "Infinity|" +      // "Infinity" string
	 *
	 *       // A decimal floating-point string representing a finite positive
	 *       // number without a leading sign has at most five basic pieces:
	 *       // Digits . Digits ExponentPart FloatTypeSuffix
	 *       //
	 *       // Since this method allows integer-only strings as input
	 *       // in addition to strings of floating-point literals, the
	 *       // two sub-patterns below are simplifications of the grammar
	 *       // productions from section 3.10.2 of
	 *       // The Java Language Specification.
	 *
	 *       // Digits ._opt Digits_opt ExponentPart_opt FloatTypeSuffix_opt
	 *       "((("+Digits+"(\\.)?("+Digits+"?)("+Exp+")?)|"+
	 *
	 *       // . Digits ExponentPart_opt FloatTypeSuffix_opt
	 *       "(\\.("+Digits+")("+Exp+")?)|"+
	 *
	 *       // Hexadecimal strings
	 *       "((" +
	 *        // 0[xX] HexDigits ._opt BinaryExponent FloatTypeSuffix_opt
	 *        "(0[xX]" + HexDigits + "(\\.)?)|" +
	 *
	 *        // 0[xX] HexDigits_opt . HexDigits BinaryExponent FloatTypeSuffix_opt
	 *        "(0[xX]" + HexDigits + "?(\\.)" + HexDigits + ")" +
	 *
	 *        ")[pP][+-]?" + Digits + "))" +
	 *       "[fFdD]?))" +
	 *       "[\\x00-\\x20]*");// Optional trailing "whitespace"
	 *
	 *  if (Pattern.matches(fpRegex, myString))
	 *      Double.valueOf(myString); // Will not throw NumberFormatException
	 *  else {
	 *      // Perform suitable alternative action
	 *  }
	 * }</pre>
	 *
	 * @param      s   the string to be parsed.
	 * @return     a {@code Double} object holding the value
	 *             represented by the {@code String} argument.
	 * @throws     NumberFormatException  if the string does not contain a
	 *             parsable number.
	 */
	public static Double valueOf(String s) throws NumberFormatException {
		return new Double(parseDouble(s));
	}

	/**
	 * Returns a {@code Double} instance representing the specified
	 * {@code double} value.
	 * If a new {@code Double} instance is not required, this method
	 * should generally be used in preference to the constructor
	 * {@link #Double(double)}, as this method is likely to yield
	 * significantly better space and time performance by caching
	 * frequently requested values.
	 *
	 * Double的构造函数, 推荐重复使用该对象的链接或者缓存, 而不是每次
	 * 需要的时候进行新建, 可以带来性能和时间上的优势(如果频繁调用).
	 *
	 * @param  d a double value.
	 * @return a {@code Double} instance representing {@code d}.
	 * @since  1.5
	 */
	public static Double valueOf(double d) {
		return new Double(d);
	}

	/**
	 * Returns a new {@code double} initialized to the value
	 * represented by the specified {@code String}, as performed
	 * by the {@code valueOf} method of class
	 * {@code Double}.
	 *
	 * 将一个String转换为double值.
	 *
	 * @param  s   the string to be parsed.
	 * @return the {@code double} value represented by the string
	 *         argument.
	 * @throws NullPointerException  if the string is null
	 * @throws NumberFormatException if the string does not contain
	 *         a parsable {@code double}.
	 * @see    java.lang.Double#valueOf(String)
	 * @since 1.2
	 */
	public static double parseDouble(String s) throws NumberFormatException {
		return FloatingDecimal.parseDouble(s);
	}

	/**
	 * Returns {@code true} if the specified number is a
	 * Not-a-Number (NaN) value, {@code false} otherwise.
	 *
	 * 判断一个数是否是NaN值.
	 *
	 * @param   v   the value to be tested.
	 * @return  {@code true} if the value of the argument is NaN;
	 *          {@code false} otherwise.
	 */
	public static boolean isNaN(double v) {
		return (v != v);
	}

	/**
	 * Returns {@code true} if the specified number is infinitely
	 * large in magnitude, {@code false} otherwise.
	 * 判断一个浮点数是否是无穷大.
	 * @param   v   the value to be tested.
	 * @return  {@code true} if the value of the argument is positive
	 *          infinity or negative infinity; {@code false} otherwise.
	 */
	public static boolean isInfinite(double v) {
		return (v == POSITIVE_INFINITY) || (v == NEGATIVE_INFINITY);
	}

	/**
	 * Returns {@code true} if the argument is a finite floating-point
	 * value; returns {@code false} otherwise (for NaN and infinity
	 * arguments).
	 * 	判断一个浮点数是否是有穷大.
	 * @param d the {@code double} value to be tested
	 * @return {@code true} if the argument is a finite
	 * floating-point value, {@code false} otherwise.
	 * @since 1.8
	 */
	public static boolean isFinite(double d) {
		return Math.abs(d) <= DoubleConsts.MAX_VALUE;
	}

	/**
	 * The value of the Double.
	 * 封装的double值
	 * @serial
	 */
	private final double value;

	/**
	 * Constructs a newly allocated {@code Double} object that
	 * represents the primitive {@code double} argument.
	 * 构造函数
	 * @param   value   the value to be represented by the {@code Double}.
	 */
	public Double(double value) {
		this.value = value;
	}

	/**
	 * Constructs a newly allocated {@code Double} object that
	 * represents the floating-point value of type {@code double}
	 * represented by the string. The string is converted to a
	 * {@code double} value as if by the {@code valueOf} method.
	 * 构造函数(string)
	 * @param  s  a string to be converted to a {@code Double}.
	 * @throws    NumberFormatException  if the string does not contain a
	 *            parsable number.
	 * @see       java.lang.Double#valueOf(java.lang.String)
	 */
	public Double(String s) throws NumberFormatException {
		value = parseDouble(s);
	}

	/**
	 * Returns {@code true} if this {@code Double} value is
	 * a Not-a-Number (NaN), {@code false} otherwise.
	 * 是否是NaN值.
	 * @return  {@code true} if the value represented by this object is
	 *          NaN; {@code false} otherwise.
	 */
	public boolean isNaN() {
		return isNaN(value);
	}

	/**
	 * Returns {@code true} if this {@code Double} value is
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
	 * Returns a string representation of this {@code Double} object.
	 * The primitive {@code double} value represented by this
	 * object is converted to a string exactly as if by the method
	 * {@code toString} of one argument.
	 * 转换为String对象.
	 * @return  a {@code String} representation of this object.
	 * @see java.lang.Double#toString(double)
	 */
	public String toString() {
		return toString(value);
	}

	//以下为Number接口的实现方法,全为窄转换可能丢失精度

	/**
	 * Returns the value of this {@code Double} as a {@code byte}
	 * after a narrowing primitive conversion.
	 *
	 * @return  the {@code double} value represented by this object
	 *          converted to type {@code byte}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 * @since JDK1.1
	 */
	public byte byteValue() {
		return (byte)value;
	}

	/**
	 * Returns the value of this {@code Double} as a {@code short}
	 * after a narrowing primitive conversion.
	 *
	 * @return  the {@code double} value represented by this object
	 *          converted to type {@code short}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 * @since JDK1.1
	 */
	public short shortValue() {
		return (short)value;
	}

	/**
	 * Returns the value of this {@code Double} as an {@code int}
	 * after a narrowing primitive conversion.
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 *
	 * @return  the {@code double} value represented by this object
	 *          converted to type {@code int}
	 */
	public int intValue() {
		return (int)value;
	}

	/**
	 * Returns the value of this {@code Double} as a {@code long}
	 * after a narrowing primitive conversion.
	 *
	 * @return  the {@code double} value represented by this object
	 *          converted to type {@code long}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public long longValue() {
		return (long)value;
	}

	/**
	 * Returns the value of this {@code Double} as a {@code float}
	 * after a narrowing primitive conversion.
	 *
	 * @return  the {@code double} value represented by this object
	 *          converted to type {@code float}
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 * @since JDK1.0
	 */
	public float floatValue() {
		return (float)value;
	}

	/**
	 * Returns the {@code double} value of this {@code Double} object.
	 *
	 * @return the {@code double} value represented by this object
	 */
	public double doubleValue() {
		return value;
	}

	/**
	 * Returns a hash code for this {@code Double} object. The
	 * result is the exclusive OR of the two halves of the
	 * {@code long} integer bit representation, exactly as
	 * produced by the method {@link #doubleToLongBits(double)}, of
	 * the primitive {@code double} value represented by this
	 * {@code Double} object. That is, the hash code is the value
	 * of the expression:
	 * 返回Double对象的哈希码.
	 * <blockquote>
	 *  {@code (int)(v^(v>>>32))}
	 * </blockquote>
	 *
	 * where {@code v} is defined by:
	 *
	 * <blockquote>
	 *  {@code long v = Double.doubleToLongBits(this.doubleValue());}
	 * </blockquote>
	 *
	 * @return  a {@code hash code} value for this object.
	 */
	@Override
	public int hashCode() {
		return Double.hashCode(value);
	}

	/**
	 * Returns a hash code for a {@code double} value; compatible with
	 * {@code Double.hashCode()}.
	 *
	 * Double对象的哈希码计算方法, 先转换为对应bit位的long类型.
	 * 对long类型,前后32比特位进行与操作, 转换为int.
	 * @param value the value to hash
	 * @return a hash code value for a {@code double} value.
	 * @since 1.8
	 */
	public static int hashCode(double value) {
		long bits = doubleToLongBits(value);
		return (int)(bits ^ (bits >>> 32));
	}

	/**
	 * Compares this object against the specified object.  The result
	 * is {@code true} if and only if the argument is not
	 * {@code null} and is a {@code Double} object that
	 * represents a {@code double} that has the same value as the
	 * {@code double} represented by this object. For this
	 * purpose, two {@code double} values are considered to be
	 * the same if and only if the method {@link
	 * #doubleToLongBits(double)} returns the identical
	 * {@code long} value when applied to each.
	 *
	 * <p>Note that in most cases, for two instances of class
	 * {@code Double}, {@code d1} and {@code d2}, the
	 * value of {@code d1.equals(d2)} is {@code true} if and
	 * only if
	 *
	 * <blockquote>
	 *  {@code d1.doubleValue() == d2.doubleValue()}
	 * </blockquote>
	 *
	 * <p>also has the value {@code true}. However, there are two
	 * exceptions:
	 * <ul>
	 * <li>If {@code d1} and {@code d2} both represent
	 *     {@code Double.NaN}, then the {@code equals} method
	 *     returns {@code true}, even though
	 *     {@code Double.NaN==Double.NaN} has the value
	 *     {@code false}.
	 * <li>If {@code d1} represents {@code +0.0} while
	 *     {@code d2} represents {@code -0.0}, or vice versa,
	 *     the {@code equal} test has the value {@code false},
	 *     even though {@code +0.0==-0.0} has the value {@code true}.
	 * </ul>
	 *
	 * 判断两个Double对象是否相同, 判断其内部封装的double值.
	 * This definition allows hash tables to operate properly.
	 * @param   obj   the object to compare with.
	 * @return  {@code true} if the objects are the same;
	 *          {@code false} otherwise.
	 * @see java.lang.Double#doubleToLongBits(double)
	 */
	public boolean equals(Object obj) {
		return (obj instanceof Double)
			   && (doubleToLongBits(((Double)obj).value) ==
					  doubleToLongBits(value));
	}

	/**
	 * Returns a representation of the specified floating-point value
	 * according to the IEEE 754 floating-point "double
	 * format" bit layout.
	 *
	 * <p>Bit 63 (the bit that is selected by the mask
	 * {@code 0x8000000000000000L}) represents the sign of the
	 * floating-point number. Bits
	 * 62-52 (the bits that are selected by the mask
	 * {@code 0x7ff0000000000000L}) represent the exponent. Bits 51-0
	 * (the bits that are selected by the mask
	 * {@code 0x000fffffffffffffL}) represent the significand
	 * (sometimes called the mantissa) of the floating-point number.
	 *
	 * <p>If the argument is positive infinity, the result is
	 * {@code 0x7ff0000000000000L}.
	 *
	 * <p>If the argument is negative infinity, the result is
	 * {@code 0xfff0000000000000L}.
	 *
	 * <p>If the argument is NaN, the result is
	 * {@code 0x7ff8000000000000L}.
	 *
	 * 将Double转换为对应64bit的long值. 注意这里进行了范围校验.如果超出范围
	 * 统一返回一个相同的NaN值.
	 *
	 * <p>In all cases, the result is a {@code long} integer that, when
	 * given to the {@link #longBitsToDouble(long)} method, will produce a
	 * floating-point value the same as the argument to
	 * {@code doubleToLongBits} (except all NaN values are
	 * collapsed to a single "canonical" NaN value).
	 *
	 * @param   value   a {@code double} precision floating-point number.
	 * @return the bits that represent the floating-point number.
	 */
	public static long doubleToLongBits(double value) {
		long result = doubleToRawLongBits(value);
		// Check for NaN based on values of bit fields, maximum
		// exponent and nonzero significand.
		//判断是否规范,或者超出范围.
		if ( ((result & DoubleConsts.EXP_BIT_MASK) ==
			  DoubleConsts.EXP_BIT_MASK) &&
			 (result & DoubleConsts.SIGNIF_BIT_MASK) != 0L)
			result = 0x7ff8000000000000L;
		return result;
	}

	/**
	 * Returns a representation of the specified floating-point value
	 * according to the IEEE 754 floating-point "double
	 * format" bit layout, preserving Not-a-Number (NaN) values.
	 *
	 * <p>Bit 63 (the bit that is selected by the mask
	 * {@code 0x8000000000000000L}) represents the sign of the
	 * floating-point number. Bits
	 * 62-52 (the bits that are selected by the mask
	 * {@code 0x7ff0000000000000L}) represent the exponent. Bits 51-0
	 * (the bits that are selected by the mask
	 * {@code 0x000fffffffffffffL}) represent the significand
	 * (sometimes called the mantissa) of the floating-point number.
	 *
	 * <p>If the argument is positive infinity, the result is
	 * {@code 0x7ff0000000000000L}.
	 *
	 * <p>If the argument is negative infinity, the result is
	 * {@code 0xfff0000000000000L}.
	 *
	 * <p>If the argument is NaN, the result is the {@code long}
	 * integer representing the actual NaN value.  Unlike the
	 * {@code doubleToLongBits} method,
	 * {@code doubleToRawLongBits} does not collapse all the bit
	 * patterns encoding a NaN to a single "canonical" NaN
	 * value.
	 *
	 * <p>In all cases, the result is a {@code long} integer that,
	 * when given to the {@link #longBitsToDouble(long)} method, will
	 * produce a floating-point value the same as the argument to
	 * {@code doubleToRawLongBits}.
	 *
	 * 本地方法, 将double中使用的64bit转换为对应bit位的原生long值.
	 *
	 * @param   value   a {@code double} precision floating-point number.
	 * @return the bits that represent the floating-point number.
	 * @since 1.3
	 */
	public static native long doubleToRawLongBits(double value);

	/**
	 *
	 * 将long的64bit位转换为对应位置的double值. 需要进行范围校验.
	 * 不同的NaN值需要调用Float.floatToRawIntBits进行区分, 不能直接进行判断相等.
	 * 这个方法可能返回的NaN值不是对应传递的long的bit位所对应的. 对于
	 * IEEE754中定义的quiet NaNs和signaling NaNs, 在Java中是不可区分的.
	 * 并且signaling NaNs取决于平台实现(兼容性存在问题), 并且需要在定义
	 * 的范围内(0x7ff0000000000001L-0x7fffffffffffffffL,
	 * 0xfff0000000000001L-0xffffffffffffffffL).
	 *
	 * Returns the {@code double} value corresponding to a given
	 * bit representation.
	 * The argument is considered to be a representation of a
	 * floating-point value according to the IEEE 754 floating-point
	 * "double format" bit layout.
	 *
	 * <p>If the argument is {@code 0x7ff0000000000000L}, the result
	 * is positive infinity.
	 *
	 * <p>If the argument is {@code 0xfff0000000000000L}, the result
	 * is negative infinity.
	 *
	 * <p>If the argument is any value in the range
	 * {@code 0x7ff0000000000001L} through
	 * {@code 0x7fffffffffffffffL} or in the range
	 * {@code 0xfff0000000000001L} through
	 * {@code 0xffffffffffffffffL}, the result is a NaN.  No IEEE
	 * 754 floating-point operation provided by Java can distinguish
	 * between two NaN values of the same type with different bit
	 * patterns.  Distinct values of NaN are only distinguishable by
	 * use of the {@code Double.doubleToRawLongBits} method.
	 *
	 * <p>In all other cases, let <i>s</i>, <i>e</i>, and <i>m</i> be three
	 * values that can be computed from the argument:
	 *
	 * <blockquote><pre>{@code
	 * int s = ((bits >> 63) == 0) ? 1 : -1;							//符号位
	 * int e = (int)((bits >> 52) & 0x7ffL);							//阶码位
	 * long m = (e == 0) ?
	 *                 (bits & 0xfffffffffffffL) << 1 :
	 *                 (bits & 0xfffffffffffffL) | 0x10000000000000L;	//尾数位
	 * }</pre></blockquote>
	 *
	 * Then the floating-point result equals the value of the mathematical
	 * expression <i>s</i>&middot;<i>m</i>&middot;2<sup><i>e</i>-1075</sup>.
	 *
	 * <p>Note that this method may not be able to return a
	 * {@code double} NaN with exactly same bit pattern as the
	 * {@code long} argument.  IEEE 754 distinguishes between two
	 * kinds of NaNs, quiet NaNs and <i>signaling NaNs</i>.  The
	 * differences between the two kinds of NaN are generally not
	 * visible in Java.  Arithmetic operations on signaling NaNs turn
	 * them into quiet NaNs with a different, but often similar, bit
	 * pattern.  However, on some processors merely copying a
	 * signaling NaN also performs that conversion.  In particular,
	 * copying a signaling NaN to return it to the calling method
	 * may perform this conversion.  So {@code longBitsToDouble}
	 * may not be able to return a {@code double} with a
	 * signaling NaN bit pattern.  Consequently, for some
	 * {@code long} values,
	 * {@code doubleToRawLongBits(longBitsToDouble(start))} may
	 * <i>not</i> equal {@code start}.  Moreover, which
	 * particular bit patterns represent signaling NaNs is platform
	 * dependent; although all NaN bit patterns, quiet or signaling,
	 * must be in the NaN range identified above.
	 *
	 * @param   bits   any {@code long} integer.
	 * @return  the {@code double} floating-point value with the same
	 *          bit pattern.
	 */
	public static native double longBitsToDouble(long bits);

	/**
	 * Compares two {@code Double} objects numerically.  There
	 * are two ways in which comparisons performed by this method
	 * differ from those performed by the Java language numerical
	 * comparison operators ({@code <, <=, ==, >=, >})
	 * when applied to primitive {@code double} values:
	 * <ul><li>
	 *          {@code Double.NaN} is considered by this method
	 *          to be equal to itself and greater than all other
	 *          {@code double} values (including
	 *          {@code Double.POSITIVE_INFINITY}).
	 * <li>
	 *          {@code 0.0d} is considered by this method to be greater
	 *          than {@code -0.0d}.
	 * </ul>
	 * This ensures that the <i>natural ordering</i> of
	 * {@code Double} objects imposed by this method is <i>consistent
	 * with equals</i>.
	 * Comparable接口的实现函数.等0大1小-1.
	 * @param   anotherDouble   the {@code Double} to be compared.
	 * @return  the value {@code 0} if {@code anotherDouble} is
	 *          numerically equal to this {@code Double}; a value
	 *          less than {@code 0} if this {@code Double}
	 *          is numerically less than {@code anotherDouble};
	 *          and a value greater than {@code 0} if this
	 *          {@code Double} is numerically greater than
	 *          {@code anotherDouble}.
	 *
	 * @since   1.2
	 */
	public int compareTo(Double anotherDouble) {
		return Double.compare(value, anotherDouble.value);
	}

	/**
	 * Compares the two specified {@code double} values. The sign
	 * of the integer value returned is the same as that of the
	 * integer that would be returned by the call:
	 * <pre>
	 *    new Double(d1).compareTo(new Double(d2))
	 * </pre>
	 * 静态工具方法来实现比较两个double值.等0大1小-1.
	 * @param   d1        the first {@code double} to compare
	 * @param   d2        the second {@code double} to compare
	 * @return  the value {@code 0} if {@code d1} is
	 *          numerically equal to {@code d2}; a value less than
	 *          {@code 0} if {@code d1} is numerically less than
	 *          {@code d2}; and a value greater than {@code 0}
	 *          if {@code d1} is numerically greater than
	 *          {@code d2}.
	 * @since 1.4
	 */
	public static int compare(double d1, double d2) {
		if (d1 < d2)
			return -1;           // Neither val is NaN, thisVal is smaller
		if (d1 > d2)
			return 1;            // Neither val is NaN, thisVal is larger

		// Cannot use doubleToRawLongBits because of possibility of NaNs.
		long thisBits    = Double.doubleToLongBits(d1);
		long anotherBits = Double.doubleToLongBits(d2);

		return (thisBits == anotherBits ?  0 : // Values are equal
				(thisBits < anotherBits ? -1 : // (-0.0, 0.0) or (!NaN, NaN)
				 1));                          // (0.0, -0.0) or (NaN, !NaN)
	}

	/**
	 * Adds two {@code double} values together as per the + operator.
	 * 求和
	 * @param a the first operand
	 * @param b the second operand
	 * @return the sum of {@code a} and {@code b}
	 * @jls 4.2.4 Floating-Point Operations
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static double sum(double a, double b) {
		return a + b;
	}

	/**
	 * Returns the greater of two {@code double} values
	 * as if by calling {@link Math#max(double, double) Math.max}.
	 * 求较大值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the greater of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static double max(double a, double b) {
		return Math.max(a, b);
	}

	/**
	 * Returns the smaller of two {@code double} values
	 * as if by calling {@link Math#min(double, double) Math.min}.
	 * 求较小值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the smaller of {@code a} and {@code b}.
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static double min(double a, double b) {
		return Math.min(a, b);
	}

	/** use serialVersionUID from JDK 1.0.2 for interoperability */
	private static final long serialVersionUID = -9172774392245257468L;
}
```