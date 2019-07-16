# Long

```java
package java.lang;
/**
 * The {@code Long} class wraps a value of the primitive type {@code
 * long} in an object. An object of type {@code Long} contains a
 * single field whose type is {@code long}.
 *
 * <p> In addition, this class provides several methods for converting
 * a {@code long} to a {@code String} and a {@code String} to a {@code
 * long}, as well as other constants and methods useful when dealing
 * with a {@code long}.
 *
 * <p>Implementation note: The implementations of the "bit twiddling"
 * methods (such as {@link #highestOneBit(long) highestOneBit} and
 * {@link #numberOfTrailingZeros(long) numberOfTrailingZeros}) are
 * based on material from Henry S. Warren, Jr.'s <i>Hacker's
 * Delight</i>, (Addison Wesley, 2002).
 *
 * Long类封装了原始类型long值, 另外Long类中提供一些String转换和位运算的
 * 工具类方法。
 *
 * @author  Lee Boynton
 * @author  Arthur van Hoff
 * @author  Josh Bloch
 * @author  Joseph D. Darcy
 * @since   JDK1.0
 */
public final class Long extends Number implements Comparable<Long> {
	/**
	 * A constant holding the minimum value a {@code long} can
	 * have, -2<sup>63</sup>.
	 *
	 * 最小值 - 2^63, 64位二进制数，第一位为符号位。
	 */
	@Native public static final long MIN_VALUE = 0x8000000000000000L;

	/**
	 * A constant holding the maximum value a {@code long} can
	 * have, 2<sup>63</sup>-1.
	 *
	 * 最大值 2^63 - 1
	 */
	@Native public static final long MAX_VALUE = 0x7fffffffffffffffL;

	/**
	 * The {@code Class} instance representing the primitive type
	 * {@code long}.
	 *
	 * 原始类型long在JVM中定义的Class对象
	 * @since   JDK1.1
	 */
	@SuppressWarnings("unchecked")
	public static final Class<Long>     TYPE = (Class<Long>) Class.getPrimitiveClass("long");

	/**
	 * Returns a string representation of the first argument in the
	 * radix specified by the second argument.
	 *
	 * <p>If the radix is smaller than {@code Character.MIN_RADIX}
	 * or larger than {@code Character.MAX_RADIX}, then the radix
	 * {@code 10} is used instead.
	 *
	 * <p>If the first argument is negative, the first element of the
	 * result is the ASCII minus sign {@code '-'}
	 * ({@code '\u005Cu002d'}). If the first argument is not
	 * negative, no sign character appears in the result.
	 * 
	 *
	 * <p>The remaining characters of the result represent the magnitude
	 * of the first argument. If the magnitude is zero, it is
	 * represented by a single zero character {@code '0'}
	 * ({@code '\u005Cu0030'}); otherwise, the first character of
	 * the representation of the magnitude will not be the zero
	 * character.  The following ASCII characters are used as digits:
	 *
	 * 返回的string类型,第一位要么为负号(-), 要么为第一位非0的数字(正数). 
	 *
	 *  下面为进制中可能使用的字母：
	 * <blockquote>
	 *   {@code 0123456789abcdefghijklmnopqrstuvwxyz}
	 * </blockquote>
	 *
	 * These are {@code '\u005Cu0030'} through
	 * {@code '\u005Cu0039'} and {@code '\u005Cu0061'} through
	 * {@code '\u005Cu007a'}. If {@code radix} is
	 * <var>N</var>, then the first <var>N</var> of these characters
	 * are used as radix-<var>N</var> digits in the order shown. Thus,
	 * the digits for hexadecimal (radix 16) are
	 * {@code 0123456789abcdef}. If uppercase letters are
	 * desired, the {@link java.lang.String#toUpperCase()} method may
	 * be called on the result:
	 *
	 * <blockquote>
	 *  {@code Long.toString(n, 16).toUpperCase()}
	 * </blockquote>
	 *
	 * 默认使用小写的字母代表进制，如果需要大写的字母，使用toUpperCase()方法。
	 *
	 * @param   i       a {@code long} to be converted to a string.
	 * @param   radix   the radix to use in the string representation.
	 * @return  a string representation of the argument in the specified radix.
	 * @see     java.lang.Character#MAX_RADIX
	 * @see     java.lang.Character#MIN_RADIX
	 */
	public static String toString(long i, int radix) {
		//如果进制小于2,或者大于36就使用默认的10进制.
		if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
			radix = 10;
		if (radix == 10) //使用快速版本,进行转换
			return toString(i);
		//新建数组进行存储，最长为65字符(二进制的时候, 一位符号)
		char[] buf = new char[65];
		int charPos = 64;
		boolean negative = (i < 0);

		if (!negative) {
			i = -i; 	//i取负值
		}

		//相当于i的绝对值要大于radix ==> (|i| > radix)
		//从高位写到低位
		while (i <= -radix) {
			buf[charPos--] = Integer.digits[(int)(-(i % radix))];
			i = i / radix;
		}
		//存储第一位数字
		buf[charPos] = Integer.digits[(int)(-i)];
		//存储符号位
		if (negative) {
			buf[--charPos] = '-';
		}

		return new String(buf, charPos, (65 - charPos));
	}

	/**
	 * Returns a string representation of the first argument as an
	 * unsigned integer value in the radix specified by the second
	 * argument.
	 *
	 * <p>If the radix is smaller than {@code Character.MIN_RADIX}
	 * or larger than {@code Character.MAX_RADIX}, then the radix
	 * {@code 10} is used instead.
	 *
	 * <p>Note that since the first argument is treated as an unsigned
	 * value, no leading sign character is printed.
	 *
	 * 由于是无符号数， 默认是正数， 不会答应符号位
	 *
	 * <p>If the magnitude is zero, it is represented by a single zero
	 * character {@code '0'} ({@code '\u005Cu0030'}); otherwise,
	 * the first character of the representation of the magnitude will
	 * not be the zero character.
	 *
	 * <p>The behavior of radixes and the characters used as digits
	 * are the same as {@link #toString(long, int) toString}.
	 *
	 * @param   i       an integer to be converted to an unsigned string.
	 * @param   radix   the radix to use in the string representation.
	 * @return  an unsigned string representation of the argument in the specified radix.
	 * @see     #toString(long, int)
	 * @since 1.8
	 */
	public static String toUnsignedString(long i, int radix) {
		if (i >= 0) //如果是正数，调用toString()方法进行转换
			return toString(i, radix);
		else { //负数的情况
			switch (radix) {
			case 2:
				return toBinaryString(i);

			case 4:
				return toUnsignedString0(i, 2);

			case 8:
				return toOctalString(i);

			case 10:
				/*
				 * We can get the effect of an unsigned division by 10
				 * on a long value by first shifting right, yielding a
				 * positive value, and then dividing by 5.  This
				 * allows the last digit and preceding digits to be
				 * isolated more quickly than by an initial conversion
				 * to BigInteger.
				 *
				 * 通过无符号右移(取正并除以2), 再除以5, 相当于除以了10(无符号值除以10).
				 * 然后取出被除以10省略的个位, 使用toString方法获取(除以10的正数)并接
				 * 个位数,形成最后的结果
				 */
				long quot = (i >>> 1) / 5;
				long rem = i - quot * 10;
				return toString(quot) + rem;

			case 16:
				return toHexString(i);

			case 32:
				return toUnsignedString0(i, 5);

			default:
				return toUnsignedBigInteger(i).toString(radix);
			}
		}
	}

	/**
	 * Return a BigInteger equal to the unsigned value of the
	 * argument.
	 *
	 * 返回一个对应无符号值的BigInteger对象
	 */
	private static BigInteger toUnsignedBigInteger(long i) {
		if (i >= 0L) //如果是正数,直接调用valueOf
			return BigInteger.valueOf(i);
		else {  //负数转换为两位int进行相加
			int upper = (int) (i >>> 32);
			int lower = (int) i;
			//保证非负数,调用Integer.toUnsignedLong进行转换
			// return (upper << 32) + lower
			return (BigInteger.valueOf(Integer.toUnsignedLong(upper))).shiftLeft(32).
				add(BigInteger.valueOf(Integer.toUnsignedLong(lower)));
		}
	}

	/**
	 * Returns a string representation of the {@code long}
	 * argument as an unsigned integer in base&nbsp;16.
	 *
	 * 返回16进制的无符号数字字符串表示
	 *
	 * <p>The unsigned {@code long} value is the argument plus
	 * 2<sup>64</sup> if the argument is negative; otherwise, it is
	 * equal to the argument.  This value is converted to a string of
	 * ASCII digits in hexadecimal (base&nbsp;16) with no extra
	 * leading {@code 0}s.
	 *
	 * 如果传递的long值是负值, 那对应的无符号值为该值加上2^64.
	 *
	 * <p>The value of the argument can be recovered from the returned
	 * string {@code s} by calling {@link
	 * Long#parseUnsignedLong(String, int) Long.parseUnsignedLong(s,
	 * 16)}.
	 *
	 * <p>If the unsigned magnitude is zero, it is represented by a
	 * single zero character {@code '0'} ({@code '\u005Cu0030'});
	 * otherwise, the first character of the representation of the
	 * unsigned magnitude will not be the zero character. The
	 * following characters are used as hexadecimal digits:
	 *
	 * <blockquote>
	 *  {@code 0123456789abcdef}
	 * </blockquote>
	 *
	 * These are the characters {@code '\u005Cu0030'} through
	 * {@code '\u005Cu0039'} and  {@code '\u005Cu0061'} through
	 * {@code '\u005Cu0066'}.  If uppercase letters are desired,
	 * the {@link java.lang.String#toUpperCase()} method may be called
	 * on the result:
	 *
	 * <blockquote>
	 *  {@code Long.toHexString(n).toUpperCase()}
	 * </blockquote>
	 *
	 * @param   i   a {@code long} to be converted to a string.
	 * @return  the string representation of the unsigned {@code long}
	 *          value represented by the argument in hexadecimal
	 *          (base&nbsp;16).
	 * @see #parseUnsignedLong(String, int)
	 * @see #toUnsignedString(long, int)
	 * @since   JDK 1.0.2
	 */
	public static String toHexString(long i) {
		return toUnsignedString0(i, 4);
	}

	/**
	 * Returns a string representation of the {@code long}
	 * argument as an unsigned integer in base&nbsp;2.
	 *
	 * 返回二进制表示的无符号值
	 * 
	 * <p>The unsigned {@code long} value is the argument plus
	 * 2<sup>64</sup> if the argument is negative; otherwise, it is
	 * equal to the argument.  This value is converted to a string of
	 * ASCII digits in binary (base&nbsp;2) with no extra leading
	 * {@code 0}s.
	 *
	 * 如果传递的long值是负值, 那对应的无符号值为该值加上2^64.
	 *
	 * <p>The value of the argument can be recovered from the returned
	 * string {@code s} by calling {@link
	 * Long#parseUnsignedLong(String, int) Long.parseUnsignedLong(s,
	 * 2)}.
	 *
	 * <p>If the unsigned magnitude is zero, it is represented by a
	 * single zero character {@code '0'} ({@code '\u005Cu0030'});
	 * otherwise, the first character of the representation of the
	 * unsigned magnitude will not be the zero character. The
	 * characters {@code '0'} ({@code '\u005Cu0030'}) and {@code
	 * '1'} ({@code '\u005Cu0031'}) are used as binary digits.
	 *
	 * @param   i   a {@code long} to be converted to a string.
	 * @return  the string representation of the unsigned {@code long}
	 *          value represented by the argument in binary (base&nbsp;2).
	 * @see #parseUnsignedLong(String, int)
	 * @see #toUnsignedString(long, int)
	 * @since   JDK 1.0.2
	 */
	public static String toBinaryString(long i) {
		return toUnsignedString0(i, 1);
	}

	/**
	 * 格式化对应的字符串, shift为进制(值为2^shift), 1 ==> 2进制
	 * 
	 * Format a long (treated as unsigned) into a String.
	 * @param val the value to format
	 * @param shift the log2 of the base to format in (4 for hex, 3 for octal, 1 for binary)
	 */
	static String toUnsignedString0(long val, int shift) {
		// assert shift > 0 && shift <=5 : "Illegal shift value"; 一般不允许超过32进制
		//获取最高位非0的位数(即二进制中的有效位数)
		int mag = Long.SIZE - Long.numberOfLeadingZeros(val);
		//mag + (shift - 1) : 对位数进行补充. 1) 如果mag是shift的倍数, shift - 1 并不会多出一位
		//2)如果不是shift倍数, 通过添加shift - 1, 保证可以提升一位, 正好是我们想要的. 类似向
		//上取整.  取Math.max则是为了防止mag为0, 即val为0的情况,保证一位来存储0.
		int chars = Math.max(((mag + (shift - 1)) / shift), 1);
		char[] buf = new char[chars];
		//格式化对象, 并存储在char数组里
		formatUnsignedLong(val, shift, buf, 0, chars);
		return new String(buf, true);
	}

	/**
	 * Format a long (treated as unsigned) into a character buffer.
	 * @param val the unsigned long to format
	 * @param shift the log2 of the base to format in (4 for hex, 3 for octal, 1 for binary)
	 * @param buf the character buffer to write to
	 * @param offset the offset in the destination buffer to start at
	 * @param len the number of characters to write
	 * @return the lowest character location used
	 */
	 static int formatUnsignedLong(long val, int shift, char[] buf, int offset, int len) {
		int charPos = len;
		int radix = 1 << shift;  //相当于2^shift, 如3: radix = 2 ^ 3 = 8
		int mask = radix - 1;    // 0111 ..., 第一位为0, 余下shift位为1, 如3时为: 0111
		do {
			//每次取shift位2进制数, 通过&mask保证取最后shift位,并转换为int取出对应的字符
			buf[offset + --charPos] = Integer.digits[((int) val) & mask];
			//无符号右移shift位，去后面位数
			val >>>= shift;
		} while (val != 0 && charPos > 0);

		return charPos;
	}

	/**
	 * Returns a {@code String} object representing the specified
	 * {@code long}.  The argument is converted to signed decimal
	 * representation and returned as a string, exactly as if the
	 * argument and the radix 10 were given as arguments to the {@link
	 * #toString(long, int)} method.
	 *
	 * 返回long对象的String, 默认使用十进制带符号标识
	 *
	 * @param   i   a {@code long} to be converted.
	 * @return  a string representation of the argument in base&nbsp;10.
	 */
	public static String toString(long i) {
		if (i == Long.MIN_VALUE)
			return "-9223372036854775808";
		int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
		char[] buf = new char[size];
		getChars(i, size, buf);
		return new String(buf, true);
	}

	/**
	 * Returns a string representation of the argument as an unsigned
	 * decimal value.
	 *
	 * 返回无符号的long的String表达式, 默认使用10进制.
	 *
	 * The argument is converted to unsigned decimal representation
	 * and returned as a string exactly as if the argument and radix
	 * 10 were given as arguments to the {@link #toUnsignedString(long,
	 * int)} method.
	 *
	 * @param   i  an integer to be converted to an unsigned string.
	 * @return  an unsigned string representation of the argument.
	 * @see     #toUnsignedString(long, int)
	 * @since 1.8
	 */
	public static String toUnsignedString(long i) {
		//调用toUnsignedString(long, int)方法
		return toUnsignedString(i, 10);
	}

	/**
	 * Places characters representing the integer i into the
	 * character array buf. The characters are placed into
	 * the buffer backwards starting with the least significant
	 * digit at the specified index (exclusive), and working
	 * backwards from there.
	 *
	 * Will fail if i == Long.MIN_VALUE
	 *
	 * 将i的显示字符从位置大到小放入char[]数组中. 
	 * 如果i为long的最小值,则会出现问题(对最小值取负还是原来的值,
	 * 在后面数组值读取时会越界)
	 *
	 */
	static void getChars(long i, int index, char[] buf) {
		long q;
		int r;
		int charPos = index;
		char sign = 0;

		if (i < 0) { //保证i为正
			sign = '-';
			i = -i;
		}

		// Get 2 digits/iteration using longs until quotient fits into an int
		//每次取后两位数(10进制), 知道i <= Intger的最大值 2 ^ 32 - 1
		while (i > Integer.MAX_VALUE) {
			q = i / 100;
			// really: r = i - (q * 100);
			//相当于 r = i - (q * 64 + q * 32 + q * 4) = i - q * 100
			//r取的值就为i的后两位(十位和个位)	
			r = (int)(i - ((q << 6) + (q << 5) + (q << 2)));
			i = q;
			buf[--charPos] = Integer.DigitOnes[r];
			buf[--charPos] = Integer.DigitTens[r];
		}

		// Get 2 digits/iteration using ints
		//每次取后两位数(10进制), 知道i < Intger的最大值 65536 (2^16)
		int q2;
		int i2 = (int)i;
		while (i2 >= 65536) {
			q2 = i2 / 100;
			// really: r = i2 - (q * 100);
			r = i2 - ((q2 << 6) + (q2 << 5) + (q2 << 2));
			i2 = q2;
			buf[--charPos] = Integer.DigitOnes[r];
			buf[--charPos] = Integer.DigitTens[r];
		}

		// Fall thru to fast mode for smaller numbers
		// assert(i2 <= 65536, i2);
		//每次去1位数进行转换
		for (;;) {
			//相当于 q = i2 * (52429 / 2 ^ 19) = i2 * 0.1000003
			q2 = (i2 * 52429) >>> (16+3);
			//相当于 r = i2 - q2 * 10 ,取得值为个位数
			r = i2 - ((q2 << 3) + (q2 << 1));  // r = i2-(q2*10) ...
			buf[--charPos] = Integer.digits[r];
			i2 = q2;
			if (i2 == 0) break;
		}
		if (sign != 0) {
			buf[--charPos] = sign;
		}
	}

	// Requires positive x
	//获取long的字符大小
	static int stringSize(long x) {
		long p = 10;
		for (int i=1; i<19; i++) {
			if (x < p)
				return i;
			p = 10*p;
		}
		return 19;
	}

	/**
	 * Parses the string argument as a signed {@code long} in the
	 * radix specified by the second argument. The characters in the
	 * string must all be digits of the specified radix (as determined
	 * by whether {@link java.lang.Character#digit(char, int)} returns
	 * a nonnegative value), except that the first character may be an
	 * ASCII minus sign {@code '-'} ({@code '\u005Cu002D'}) to
	 * indicate a negative value or an ASCII plus sign {@code '+'}
	 * ({@code '\u005Cu002B'}) to indicate a positive value. The
	 * resulting {@code long} value is returned.
	 *
	 * 按照对应的进制（radix），将string转换为long.
	 *
	 * <p>Note that neither the character {@code L}
	 * ({@code '\u005Cu004C'}) nor {@code l}
	 * ({@code '\u005Cu006C'}) is permitted to appear at the end
	 * of the string as a type indicator, as would be permitted in
	 * Java programming language source code - except that either
	 * {@code L} or {@code l} may appear as a digit for a
	 * radix greater than or equal to 22.
	 *
	 * <p>An exception of type {@code NumberFormatException} is
	 * thrown if any of the following situations occurs:
	 * <ul>
	 *
	 * <li>The first argument is {@code null} or is a string of
	 * length zero.
	 *
	 * <li>The {@code radix} is either smaller than {@link
	 * java.lang.Character#MIN_RADIX} or larger than {@link
	 * java.lang.Character#MAX_RADIX}.
	 *
	 * <li>Any character of the string is not a digit of the specified
	 * radix, except that the first character may be a minus sign
	 * {@code '-'} ({@code '\u005Cu002d'}) or plus sign {@code
	 * '+'} ({@code '\u005Cu002B'}) provided that the string is
	 * longer than length 1.
	 *
	 * <li>The value represented by the string is not a value of type
	 *      {@code long}.
	 * </ul>
	 *
	 * <p>Examples:
	 * <blockquote><pre>
	 * parseLong("0", 10) returns 0L
	 * parseLong("473", 10) returns 473L
	 * parseLong("+42", 10) returns 42L
	 * parseLong("-0", 10) returns 0L
	 * parseLong("-FF", 16) returns -255L
	 * parseLong("1100110", 2) returns 102L
	 * parseLong("99", 8) throws a NumberFormatException
	 * parseLong("Hazelnut", 10) throws a NumberFormatException
	 * parseLong("Hazelnut", 36) returns 1356099454469L
	 * </pre></blockquote>
	 *
	 * @param      s       the {@code String} containing the
	 *                     {@code long} representation to be parsed.
	 * @param      radix   the radix to be used while parsing {@code s}.
	 * @return     the {@code long} represented by the string argument in
	 *             the specified radix.
	 * @throws     NumberFormatException  if the string does not contain a
	 *             parsable {@code long}.
	 */
	public static long parseLong(String s, int radix)
			  throws NumberFormatException
	{
		if (s == null) {
			//传递的string对象不能为null
			throw new NumberFormatException("null");
		}

		if (radix < Character.MIN_RADIX) {
			//传递的radix不能小于2
			throw new NumberFormatException("radix " + radix +
											" less than Character.MIN_RADIX");
		}
		if (radix > Character.MAX_RADIX) {
			//传递的radix不能大于36
			throw new NumberFormatException("radix " + radix +
											" greater than Character.MAX_RADIX");
		}

		long result = 0;
		boolean negative = false;
		int i = 0, len = s.length();
		long limit = -Long.MAX_VALUE;
		long multmin;
		int digit;

		if (len > 0) {
			char firstChar = s.charAt(0);
			if (firstChar < '0') { // Possible leading "+" or "-"
				if (firstChar == '-') {
					negative = true;
					limit = Long.MIN_VALUE;
				} else if (firstChar != '+')
					throw NumberFormatException.forInputString(s);
				//第一位字符如果不是数字,只能是+,-
				
				if (len == 1) // Cannot have lone "+" or "-"
					throw NumberFormatException.forInputString(s);
				i++;
			}
			multmin = limit / radix;
			while (i < len) {
				// Accumulating negatively avoids surprises near MAX_VALUE
				// 这里进行累计是使用负数的形式（累加使用减法）
				
				//使用Character.digit()来获取对应字符所代表的值(不在范围内范围-1)
				digit = Character.digit(s.charAt(i++),radix);
				if (digit < 0) { //该位数必须大于0
					throw NumberFormatException.forInputString(s);
				}
				if (result < multmin) { //该位数不能超过了最大值
					throw NumberFormatException.forInputString(s);
				}
				result *= radix;
				if (result < limit + digit) { //判断加上digit之后是否超出了范围
					throw NumberFormatException.forInputString(s);
				}
				//负数,使用减法进行累加
				result -= digit;
			}
		} else {
			throw NumberFormatException.forInputString(s);
		}
		return negative ? result : -result;
	}

	/**
	 * Parses the string argument as a signed decimal {@code long}.
	 * The characters in the string must all be decimal digits, except
	 * that the first character may be an ASCII minus sign {@code '-'}
	 * ({@code \u005Cu002D'}) to indicate a negative value or an
	 * ASCII plus sign {@code '+'} ({@code '\u005Cu002B'}) to
	 * indicate a positive value. The resulting {@code long} value is
	 * returned, exactly as if the argument and the radix {@code 10}
	 * were given as arguments to the {@link
	 * #parseLong(java.lang.String, int)} method.
	 *
	 * 默认10进制，调用上面的方法进行转换
	 *
	 * <p>Note that neither the character {@code L}
	 * ({@code '\u005Cu004C'}) nor {@code l}
	 * ({@code '\u005Cu006C'}) is permitted to appear at the end
	 * of the string as a type indicator, as would be permitted in
	 * Java programming language source code.
	 *
	 * @param      s   a {@code String} containing the {@code long}
	 *             representation to be parsed
	 * @return     the {@code long} represented by the argument in
	 *             decimal.
	 * @throws     NumberFormatException  if the string does not contain a
	 *             parsable {@code long}.
	 */
	public static long parseLong(String s) throws NumberFormatException {
		return parseLong(s, 10);
	}

	/**
	 * Parses the string argument as an unsigned {@code long} in the
	 * radix specified by the second argument.  An unsigned integer
	 * maps the values usually associated with negative numbers to
	 * positive numbers larger than {@code MAX_VALUE}.
	 *
	 * 将一个string对象转换为一个无符号long类型
	 *
	 * The characters in the string must all be digits of the
	 * specified radix (as determined by whether {@link
	 * java.lang.Character#digit(char, int)} returns a nonnegative
	 * value), except that the first character may be an ASCII plus
	 * sign {@code '+'} ({@code '\u005Cu002B'}). The resulting
	 * integer value is returned.
	 *
	 * 无符号数字不能允许负数，第一位非数字进制数只能为+。
	 *
	 * <p>An exception of type {@code NumberFormatException} is
	 * thrown if any of the following situations occurs:
	 * <ul>
	 * <li>The first argument is {@code null} or is a string of
	 * length zero.
	 *
	 * <li>The radix is either smaller than
	 * {@link java.lang.Character#MIN_RADIX} or
	 * larger than {@link java.lang.Character#MAX_RADIX}.
	 *
	 * <li>Any character of the string is not a digit of the specified
	 * radix, except that the first character may be a plus sign
	 * {@code '+'} ({@code '\u005Cu002B'}) provided that the
	 * string is longer than length 1.
	 *
	 * <li>The value represented by the string is larger than the
	 * largest unsigned {@code long}, 2<sup>64</sup>-1.
	 *
	 * long类型的无符号数字的最大值为： 2^64 - 1.(有符号： 2^63 - 1)
	 *
	 * </ul>
	 *
	 *
	 * @param      s   the {@code String} containing the unsigned integer
	 *                  representation to be parsed
	 * @param      radix   the radix to be used while parsing {@code s}.
	 * @return     the unsigned {@code long} represented by the string
	 *             argument in the specified radix.
	 * @throws     NumberFormatException if the {@code String}
	 *             does not contain a parsable {@code long}.
	 * @since 1.8
	 */
	public static long parseUnsignedLong(String s, int radix)
				throws NumberFormatException {
		if (s == null)  {
			throw new NumberFormatException("null");
		}

		int len = s.length();
		if (len > 0) {
			char firstChar = s.charAt(0);
			if (firstChar == '-') { //无符号不允许出现负数标识： -
				throw new
					NumberFormatException(String.format("Illegal leading minus sign " +
													   "on unsigned string %s.", s));
			} else {
				//如果String s的代表的数字大小不超过有符号的范围，直接调用有符号转换
				if (len <= 12 || // Long.MAX_VALUE in Character.MAX_RADIX is 13 digits, 36进制最多13位
					(radix == 10 && len <= 18) ) { // Long.MAX_VALUE in base 10 is 19 digits, 10进制19位
					return parseLong(s, radix);
				}

				// No need for range checks on len due to testing above.
				//取前len - 1位,如果可以正常转换(没有抛出跨界异常),说明前i-1位是在有符号范围内的,即2进制63位
				long first = parseLong(s.substring(0, len - 1), radix);
				//取最后一位
				int second = Character.digit(s.charAt(len - 1), radix);
				if (second < 0) { //最后一位不能越界
					throw new NumberFormatException("Bad digit at end of " + s);
				}
				//此处存疑, 如正常LONG的最大值: 7FFF_FFFF_FFFF_FFFF, 
				//对应最大的Unsigned LONG为:FFFF_FFFF_FFFF_FFFF,但是如果传递7_FFFF_FFFF_FFFF_FFFF,仍然可以正常解析
				//因为compareUnsigned为0, first,secend正常,溢出后的result和first相同.对于4_0000_0000_0000_0DEF,报错 
				//因为溢出后的result为正数, 比first小, 不能满足要求. 这里需要进一步的确定.
				//本意应该是Long的最大值对应某些进制(如3,5)无法保证最后位数一定是最大值,可能为中间值. 这时候最后一
				//位可能大于中间值,就会超出限制.
				long result = first * radix + second;
				//first可以正常转换的话, 说明first是没有越界的
				if (compareUnsigned(result, first) < 0) {
					/*
					 * The maximum unsigned value, (2^64)-1, takes at
					 * most one more digit to represent than the
					 * maximum signed value, (2^63)-1.  Therefore,
					 * parsing (len - 1) digits will be appropriately
					 * in-range of the signed parsing.  In other
					 * words, if parsing (len -1) digits overflows
					 * signed parsing, parsing len digits will
					 * certainly overflow unsigned parsing.
					 *
					 * The compareUnsigned check above catches
					 * situations where an unsigned overflow occurs
					 * incorporating the contribution of the final
					 * digit.
					 */
					throw new NumberFormatException(String.format("String value %s exceeds " +
																  "range of unsigned long.", s));
				}
				return result;
			}
		} else {
			throw NumberFormatException.forInputString(s);
		}
	}

	/**
	 * Parses the string argument as an unsigned decimal {@code long}. The
	 * characters in the string must all be decimal digits, except
	 * that the first character may be an an ASCII plus sign {@code
	 * '+'} ({@code '\u005Cu002B'}). The resulting integer value
	 * is returned, exactly as if the argument and the radix 10 were
	 * given as arguments to the {@link
	 * #parseUnsignedLong(java.lang.String, int)} method.
	 *
	 * 不传递radix，默认使用10
	 *
	 * @param s   a {@code String} containing the unsigned {@code long}
	 *            representation to be parsed
	 * @return    the unsigned {@code long} value represented by the decimal string argument
	 * @throws    NumberFormatException  if the string does not contain a
	 *            parsable unsigned integer.
	 * @since 1.8
	 */
	public static long parseUnsignedLong(String s) throws NumberFormatException {
		return parseUnsignedLong(s, 10);
	}
	
	/**
	 * Returns a {@code Long} object holding the value
	 * extracted from the specified {@code String} when parsed
	 * with the radix given by the second argument.  The first
	 * argument is interpreted as representing a signed
	 * {@code long} in the radix specified by the second
	 * argument, exactly as if the arguments were given to the {@link
	 * #parseLong(java.lang.String, int)} method. The result is a
	 * {@code Long} object that represents the {@code long}
	 * value specified by the string.
	 *
	 * <p>In other words, this method returns a {@code Long} object equal
	 * to the value of:
	 *
	 * 先调用parseLong转换为long值, 然后调用valueOf进行构造.
	 *
	 * <blockquote>
	 *  {@code new Long(Long.parseLong(s, radix))}
	 * </blockquote>
	 *
	 * @param      s       the string to be parsed
	 * @param      radix   the radix to be used in interpreting {@code s}
	 * @return     a {@code Long} object holding the value
	 *             represented by the string argument in the specified
	 *             radix.
	 * @throws     NumberFormatException  If the {@code String} does not
	 *             contain a parsable {@code long}.
	 */
	public static Long valueOf(String s, int radix) throws NumberFormatException {
		return Long.valueOf(parseLong(s, radix));
	}

    /**
     * Returns a {@code Long} object holding the value
     * of the specified {@code String}. The argument is
     * interpreted as representing a signed decimal {@code long},
     * exactly as if the argument were given to the {@link
     * #parseLong(java.lang.String)} method. The result is a
     * {@code Long} object that represents the integer value
     * specified by the string.
     *
     * <p>In other words, this method returns a {@code Long} object
     * equal to the value of:
     *
     * <blockquote>
     *  {@code new Long(Long.parseLong(s))}
     * </blockquote>
     *
	 * 不传递参数,默认传递为10.
	 *
     * @param      s   the string to be parsed.
     * @return     a {@code Long} object holding the value
     *             represented by the string argument.
     * @throws     NumberFormatException  If the string cannot be parsed
     *             as a {@code long}.
     */
    public static Long valueOf(String s) throws NumberFormatException {
        return Long.valueOf(parseLong(s, 10));
    }
	
	/**
	* Long对象的Cache,默认存储从-128到127,预先构造好对象.
	* 
	**/
	private static class LongCache {
		private LongCache(){}

		static final Long cache[] = new Long[-(-128) + 127 + 1];

		static {
			for(int i = 0; i < cache.length; i++)
				cache[i] = new Long(i - 128);
		}
	}

	/**
	 * Returns a {@code Long} instance representing the specified
	 * {@code long} value.
	 * If a new {@code Long} instance is not required, this method
	 * should generally be used in preference to the constructor
	 * {@link #Long(long)}, as this method is likely to yield
	 * significantly better space and time performance by caching
	 * frequently requested values.
	 *
	 * Note that unlike the {@linkplain Integer#valueOf(int)
	 * corresponding method} in the {@code Integer} class, this method
	 * is <em>not</em> required to cache values within a particular
	 * range.
	 *
	 * 如果l在-128到127中, 使用缓存中对象, 否则新建对象. 优先级应该大于
	 * 构造函数.可以提高性能.
	 * 
	 * @param  l a long value.
	 * @return a {@code Long} instance representing {@code l}.
	 * @since  1.5
	 */
	public static Long valueOf(long l) {
		final int offset = 128;
		if (l >= -128 && l <= 127) { // will cache
			return LongCache.cache[(int)l + offset];
		}
		return new Long(l);
	}
	
	/**
	 * 解密String对象为Long对象，接收十进制，十六进制，八进制
	 *
	 * Decodes a {@code String} into a {@code Long}.
	 * Accepts decimal, hexadecimal, and octal numbers given by the
	 * following grammar:
	 *
	 * <blockquote>
	 * <dl>
	 * <dt><i>DecodableString:</i>
	 * <dd><i>Sign<sub>opt</sub> DecimalNumeral</i>
	 * <dd><i>Sign<sub>opt</sub></i> {@code 0x} <i>HexDigits</i>
	 * <dd><i>Sign<sub>opt</sub></i> {@code 0X} <i>HexDigits</i>
	 * <dd><i>Sign<sub>opt</sub></i> {@code #} <i>HexDigits</i>
	 * <dd><i>Sign<sub>opt</sub></i> {@code 0} <i>OctalDigits</i>
	 *
	 * 0x,0X,#代表十六进制，0代表八进制，默认则是10进制，接收正负符号
	 *
	 * <dt><i>Sign:</i>
	 * <dd>{@code -}
	 * <dd>{@code +}
	 * </dl>
	 * </blockquote>
	 *
	 * <i>DecimalNumeral</i>, <i>HexDigits</i>, and <i>OctalDigits</i>
	 * are as defined in section 3.10.1 of
	 * <cite>The Java&trade; Language Specification</cite>,
	 * except that underscores are not accepted between digits.
	 *
	 * <p>The sequence of characters following an optional
	 * sign and/or radix specifier ("{@code 0x}", "{@code 0X}",
	 * "{@code #}", or leading zero) is parsed as by the {@code
	 * Long.parseLong} method with the indicated radix (10, 16, or 8).
	 * This sequence of characters must represent a positive value or
	 * a {@link NumberFormatException} will be thrown.  The result is
	 * negated if first character of the specified {@code String} is
	 * the minus sign.  No whitespace characters are permitted in the
	 * {@code String}.
	 *
	 * @param     nm the {@code String} to decode.
	 * @return    a {@code Long} object holding the {@code long}
	 *            value represented by {@code nm}
	 * @throws    NumberFormatException  if the {@code String} does not
	 *            contain a parsable {@code long}.
	 * @see java.lang.Long#parseLong(String, int)
	 * @since 1.2
	 */
	public static Long decode(String nm) throws NumberFormatException {
		int radix = 10;
		int index = 0;
		boolean negative = false;
		Long result;

		if (nm.length() == 0)
			throw new NumberFormatException("Zero length string");
		char firstChar = nm.charAt(0);
		// Handle sign, if present
		if (firstChar == '-') {
			negative = true;
			index++;
		} else if (firstChar == '+')
			index++;

		// Handle radix specifier, if present
		if (nm.startsWith("0x", index) || nm.startsWith("0X", index)) {
			index += 2;
			radix = 16;
		}
		else if (nm.startsWith("#", index)) {
			index ++;
			radix = 16;
		}
		else if (nm.startsWith("0", index) && nm.length() > 1 + index) {
			index ++;
			radix = 8;
		}
		
		//出现+/-/0x/0X/#/0之后不允许再添加+，-
		if (nm.startsWith("-", index) || nm.startsWith("+", index))
			throw new NumberFormatException("Sign character in wrong position");
		//调用Long.valueOf进行转换
		try {
			result = Long.valueOf(nm.substring(index), radix);
			result = negative ? Long.valueOf(-result.longValue()) : result;
		} catch (NumberFormatException e) {
			// If number is Long.MIN_VALUE, we'll end up here. The next line
			// handles this case, and causes any genuine format error to be
			// rethrown.
			String constant = negative ? ("-" + nm.substring(index))
									   : nm.substring(index);
			result = Long.valueOf(constant, radix);
		}
		return result;
	}
	
	/**
	 * The value of the {@code Long}.
	 *
	 * @serial
	 */
	private final long value;
	
	/**
	 * Constructs a newly allocated {@code Long} object that
	 * represents the specified {@code long} argument.
	 *
	 * @param   value   the value to be represented by the
	 *          {@code Long} object.
	 */
	public Long(long value) {
		this.value = value;
	}
	
    /**
     * Constructs a newly allocated {@code Long} object that
     * represents the {@code long} value indicated by the
     * {@code String} parameter. The string is converted to a
     * {@code long} value in exactly the manner used by the
     * {@code parseLong} method for radix 10.
     *
	 * 使用10进制进行转换String
	 *
     * @param      s   the {@code String} to be converted to a
     *             {@code Long}.
     * @throws     NumberFormatException  if the {@code String} does not
     *             contain a parsable {@code long}.
     * @see        java.lang.Long#parseLong(java.lang.String, int)
     */
    public Long(String s) throws NumberFormatException {
        this.value = parseLong(s, 10);
    }
	
	//以下为Numberic接口的实现方法
	/**
	 * Returns the value of this {@code Long} as a {@code byte} after
	 * a narrowing primitive conversion. 窄转换，丢失精度
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public byte byteValue() {
		return (byte)value;
	}

	/**
	 * Returns the value of this {@code Long} as a {@code short} after
	 * a narrowing primitive conversion. 窄转换，丢失精度
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public short shortValue() {
		return (short)value;
	}

	/**
	 * Returns the value of this {@code Long} as an {@code int} after
	 * a narrowing primitive conversion. 窄转换，丢失精度
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public int intValue() {
		return (int)value;
	}

	/**
	 * Returns the value of this {@code Long} as a
	 * {@code long} value.
	 */
	public long longValue() {
		return value;
	}

	/**
	 * Returns the value of this {@code Long} as a {@code float} after
	 * a widening primitive conversion. 宽转换，不丢失精度
	 * @jls 5.1.2 Widening Primitive Conversions
	 */
	public float floatValue() {
		return (float)value;
	}

	/**
	 * Returns the value of this {@code Long} as a {@code double}
	 * after a widening primitive conversion. 宽转换，不丢失精度
	 * @jls 5.1.2 Widening Primitive Conversions
	 */
	public double doubleValue() {
		return (double)value;
	}

	/**
	 * Returns a {@code String} object representing this
	 * {@code Long}'s value.  The value is converted to signed
	 * decimal representation and returned as a string, exactly as if
	 * the {@code long} value were given as an argument to the
	 * {@link java.lang.Long#toString(long)} method.
	 *
	 * 转换为对应的String
	 *
	 * @return  a string representation of the value of this object in
	 *          base&nbsp;10.
	 */
	public String toString() {
		return toString(value);
	}
	
	/**
	 * Returns a hash code for this {@code Long}. The result is
	 * the exclusive OR of the two halves of the primitive
	 * {@code long} value held by this {@code Long}
	 * object. That is, the hashcode is the value of the expression:
	 *
	 * Long对象的HashCode计算方式: 高32位与低32位进行与操作,转换为int
	 * (取与操作之后的32位)
	 *
	 * <blockquote>
	 *  {@code (int)(this.longValue()^(this.longValue()>>>32))}
	 * </blockquote>
	 *
	 * @return  a hash code value for this object.
	 */
	@Override
	public int hashCode() {
		return Long.hashCode(value);
	}

	/**
	 * Returns a hash code for a {@code long} value; compatible with
	 * {@code Long.hashCode()}.
	 *
	 * Long对象的HashCode计算方式: 高32位与低32位进行与操作,转换为int
	 * (取与操作之后的32位)
	 * 
	 * @param value the value to hash
	 * @return a hash code value for a {@code long} value.
	 * @since 1.8
	 */
	public static int hashCode(long value) {
		return (int)(value ^ (value >>> 32));
	}
	
    /**
     * Compares this object to the specified object.  The result is
     * {@code true} if and only if the argument is not
     * {@code null} and is a {@code Long} object that
     * contains the same {@code long} value as this object.
     *
	 * 覆盖Object的equals方法, 转换为值比较
	 *
     * @param   obj   the object to compare with.
     * @return  {@code true} if the objects are the same;
     *          {@code false} otherwise.
     */
    public boolean equals(Object obj) {
        if (obj instanceof Long) {
            return value == ((Long)obj).longValue();
        }
        return false;
    }
	
	/**
	 * Determines the {@code long} value of the system property
	 * with the specified name.
	 *
	 * <p>The first argument is treated as the name of a system
	 * property.  System properties are accessible through the {@link
	 * java.lang.System#getProperty(java.lang.String)} method. The
	 * string value of this property is then interpreted as a {@code
	 * long} value using the grammar supported by {@link Long#decode decode}
	 * and a {@code Long} object representing this value is returned.
	 *
	 * <p>If there is no property with the specified name, if the
	 * specified name is empty or {@code null}, or if the property
	 * does not have the correct numeric format, then {@code null} is
	 * returned.
	 *
	 * <p>In other words, this method returns a {@code Long} object
	 * equal to the value of:
	 *
	 * 获取系统指定属性（nm），所代表的Integer值
	 * 通过调用System.getProperty(nm)获取属性，并使用Integer.decode()进行解码
	 * 如果未找到对象或为空，返回null
	 *
	 * <blockquote>
	 *  {@code getLong(nm, null)}
	 * </blockquote>
	 *
	 * @param   nm   property name.
	 * @return  the {@code Long} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     java.lang.System#getProperty(java.lang.String)
	 * @see     java.lang.System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Long getLong(String nm) {
		return getLong(nm, null);
	}

	/**
	 * Determines the {@code long} value of the system property
	 * with the specified name.
	 *
	 * <p>The first argument is treated as the name of a system
	 * property.  System properties are accessible through the {@link
	 * java.lang.System#getProperty(java.lang.String)} method. The
	 * string value of this property is then interpreted as a {@code
	 * long} value using the grammar supported by {@link Long#decode decode}
	 * and a {@code Long} object representing this value is returned.
	 *
	 * <p>The second argument is the default value. A {@code Long} object
	 * that represents the value of the second argument is returned if there
	 * is no property of the specified name, if the property does not have
	 * the correct numeric format, or if the specified name is empty or null.
	 *
	 * <p>In other words, this method returns a {@code Long} object equal
	 * to the value of:
	 *
	 * <blockquote>
	 *  {@code getLong(nm, new Long(val))}
	 * </blockquote>
	 *
	 * but in practice it may be implemented in a manner such as:
	 *
	 * <blockquote><pre>
	 * Long result = getLong(nm, null);
	 * return (result == null) ? new Long(val) : result;
	 * </pre></blockquote>
	 *
	 * to avoid the unnecessary allocation of a {@code Long} object when
	 * the default value is not needed.
	 *
	 * @param   nm    property name.
	 * @param   val   default value.
	 * @return  the {@code Long} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     java.lang.System#getProperty(java.lang.String)
	 * @see     java.lang.System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Long getLong(String nm, long val) {
		Long result = Long.getLong(nm, null);
		return (result == null) ? Long.valueOf(val) : result;
	}

	/**
	 * Returns the {@code long} value of the system property with
	 * the specified name.  The first argument is treated as the name
	 * of a system property.  System properties are accessible through
	 * the {@link java.lang.System#getProperty(java.lang.String)}
	 * method. The string value of this property is then interpreted
	 * as a {@code long} value, as per the
	 * {@link Long#decode decode} method, and a {@code Long} object
	 * representing this value is returned; in summary:
	 * 
	 * 具体实现过程.
	 * 获取系统指定属性（nm），所代表的Integer值
	 * 通过调用System.getProperty(nm)获取属性，并使用Integer.decode()进行解码
	 * 如果未找到对象或为空，返回null
	 *
	 * <ul>
	 * <li>If the property value begins with the two ASCII characters
	 * {@code 0x} or the ASCII character {@code #}, not followed by
	 * a minus sign, then the rest of it is parsed as a hexadecimal integer
	 * exactly as for the method {@link #valueOf(java.lang.String, int)}
	 * with radix 16.
	 * <li>If the property value begins with the ASCII character
	 * {@code 0} followed by another character, it is parsed as
	 * an octal integer exactly as by the method {@link
	 * #valueOf(java.lang.String, int)} with radix 8.
	 * <li>Otherwise the property value is parsed as a decimal
	 * integer exactly as by the method
	 * {@link #valueOf(java.lang.String, int)} with radix 10.
	 * </ul>
	 *
	 * <p>Note that, in every case, neither {@code L}
	 * ({@code '\u005Cu004C'}) nor {@code l}
	 * ({@code '\u005Cu006C'}) is permitted to appear at the end
	 * of the property value as a type indicator, as would be
	 * permitted in Java programming language source code.
	 *
	 * <p>The second argument is the default value. The default value is
	 * returned if there is no property of the specified name, if the
	 * property does not have the correct numeric format, or if the
	 * specified name is empty or {@code null}.
	 *
	 * @param   nm   property name.
	 * @param   val   default value.
	 * @return  the {@code Long} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     System#getProperty(java.lang.String)
	 * @see     System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Long getLong(String nm, Long val) {
		String v = null;
		try {
			v = System.getProperty(nm);
		} catch (IllegalArgumentException | NullPointerException e) {
		}
		if (v != null) {
			try {
				return Long.decode(v);
			} catch (NumberFormatException e) {
			}
		}
		return val;
	}
	
    /**
     * Compares two {@code Long} objects numerically.
     *
	 * Comparable接口的实现方法, 比较内部的值
	 *
     * @param   anotherLong   the {@code Long} to be compared.
     * @return  the value {@code 0} if this {@code Long} is
     *          equal to the argument {@code Long}; a value less than
     *          {@code 0} if this {@code Long} is numerically less
     *          than the argument {@code Long}; and a value greater
     *          than {@code 0} if this {@code Long} is numerically
     *           greater than the argument {@code Long} (signed
     *           comparison).
     * @since   1.2
     */
    public int compareTo(Long anotherLong) {
        return compare(this.value, anotherLong.value);
    }

    /**
     * Compares two {@code long} values numerically.
     * The value returned is identical to what would be returned by:
     * <pre>
     *    Long.valueOf(x).compareTo(Long.valueOf(y))
     * </pre>
     *
     * @param  x the first {@code long} to compare
     * @param  y the second {@code long} to compare
     * @return the value {@code 0} if {@code x == y};
     *         a value less than {@code 0} if {@code x < y}; and
     *         a value greater than {@code 0} if {@code x > y}
     * @since 1.7
     */
    public static int compare(long x, long y) {
        return (x < y) ? -1 : ((x == y) ? 0 : 1);
    }
	
    /**
     * Compares two {@code long} values numerically treating the values
     * as unsigned.
     *
	 * 比较无符号值. 通过比较值和MIN_VALUE值求和,进行有符号比较. 原理:
	 * 有符号时的负数的无符号值是大于正数的(第一位符号位为1)。
	 * x,y都是正数:变成负数进行比较, 没有影响. 都是负数, 这时候
	 * 通过求和,把符号位溢出为正数,比较大小, 没有影响. 一正一负,
	 * 正数补位变成负数, 负数变成正数. 负数大于正数. 正确.
	 *
     * @param  x the first {@code long} to compare
     * @param  y the second {@code long} to compare
     * @return the value {@code 0} if {@code x == y}; a value less
     *         than {@code 0} if {@code x < y} as unsigned values; and
     *         a value greater than {@code 0} if {@code x > y} as
     *         unsigned values
     * @since 1.8
     */
    public static int compareUnsigned(long x, long y) {
        return compare(x + MIN_VALUE, y + MIN_VALUE);
    }
	
	/**
	 * Returns the unsigned quotient of dividing the first argument by
	 * the second where each argument and the result is interpreted as
	 * an unsigned value.
	 *
	 * <p>Note that in two's complement arithmetic, the three other
	 * basic arithmetic operations of add, subtract, and multiply are
	 * bit-wise identical if the two operands are regarded as both
	 * being signed or both being unsigned.  Therefore separate {@code
	 * addUnsigned}, etc. methods are not provided.
	 *
	 * 对于加减乘其余3中计算, 都是按位运算的, 无论是否有符号, 不用重新定义.
	 *
	 * @param dividend the value to be divided
	 * @param divisor the value doing the dividing
	 * @return the unsigned quotient of the first argument divided by
	 * the second argument
	 * @see #remainderUnsigned
	 * @since 1.8
	 */
	public static long divideUnsigned(long dividend, long divisor) {
		if (divisor < 0L) { // signed comparison
			// Answer must be 0 or 1 depending on relative magnitude
			// of dividend and divisor.
			//如果除数是负数, 意味着第一位为1,在无符号中是非常大的,
			//这时候只要检查被除数是否大于除数,应为除数最大也不可能大于2
			return (compareUnsigned(dividend, divisor)) < 0 ? 0L :1L;
		}
		
		if (dividend > 0) //  Both inputs non-negative 除数和被除数都是大于0
			return dividend/divisor;
		else {
			/*
			 * For simple code, leveraging BigInteger.  Longer and faster
			 * code written directly in terms of operations on longs is
			 * possible; see "Hacker's Delight" for divide and remainder
			 * algorithms.
			 * 
			 * 为了简化代码,使用BigInteger进行计算,可以直接使用longs进行计算
			 * 可以参考Hacker's Delight的算法实现. 类似的还有取余算法
			 *
			 */
			return toUnsignedBigInteger(dividend).
				divide(toUnsignedBigInteger(divisor)).longValue();
		}
	}
	
	/**
	 * Returns the unsigned remainder from dividing the first argument
	 * by the second where each argument and the result is interpreted
	 * as an unsigned value.
	 *
	 * 无符号取余数
	 *
	 * @param dividend the value to be divided
	 * @param divisor the value doing the dividing
	 * @return the unsigned remainder of the first argument divided by
	 * the second argument
	 * @see #divideUnsigned
	 * @since 1.8
	 */
	public static long remainderUnsigned(long dividend, long divisor) {
		if (dividend > 0 && divisor > 0) { // signed comparisons
			return dividend % divisor;
		} else {
			//判断取余的数是否小于余数,否则调用BigInteger来实现
			if (compareUnsigned(dividend, divisor) < 0) // Avoid explicit check for 0 divisor
				return dividend;
			else
				return toUnsignedBigInteger(dividend).
					remainder(toUnsignedBigInteger(divisor)).longValue();
		}
	}
	
	// Bit Twiddling

	/**
	 * The number of bits used to represent a {@code long} value in two's
	 * complement binary form.
	 *
	 * @since 1.5
	 */
	@Native public static final int SIZE = 64;

	/**
	 * The number of bytes used to represent a {@code long} value in two's
	 * complement binary form.
	 *
	 * @since 1.8
	 */
	public static final int BYTES = SIZE / Byte.SIZE;
	
	/**
	 * Returns a {@code long} value with at most a single one-bit, in the
	 * position of the highest-order ("leftmost") one-bit in the specified
	 * {@code long} value.  Returns zero if the specified value has no
	 * one-bits in its two's complement binary representation, that is, if it
	 * is equal to zero.
	 *
	 * 返回左边最高位bit位所代表的值
	 *
	 * @param i the value whose highest one bit is to be computed
	 * @return a {@code long} value with a single one-bit, in the position
	 *     of the highest-order one-bit in the specified value, or zero if
	 *     the specified value is itself equal to zero.
	 * @since 1.5
	 */
	public static long highestOneBit(long i) {
		// HD, Figure 3-1
		i |= (i >>  1); //填充最高位右边一位为1，这是填充了1位
		i |= (i >>  2); //填充最高位右边两位为1，这时填充了3位
		i |= (i >>  4); //填充最高位右边四位为1，这时填充了7位
		i |= (i >>  8); //填充最高位右边八位为1，这时填充了15位
		i |= (i >> 16); //填充最高位右边十六位为1，这时填充了31位
		i |= (i >> 32); //填充最高位右边三十二位为1，这时填充了31位
		//现在保证了i最高位后面全为1,减法之后保证i后面全为0
		return i - (i >>> 1);
	}
	
	/**
	 * Returns a {@code long} value with at most a single one-bit, in the
	 * position of the lowest-order ("rightmost") one-bit in the specified
	 * {@code long} value.  Returns zero if the specified value has no
	 * one-bits in its two's complement binary representation, that is, if it
	 * is equal to zero.
	 *
	 * 返回左边最低位bit位所代表的值
	 *
	 * @param i the value whose lowest one bit is to be computed
	 * @return a {@code long} value with a single one-bit, in the position
	 *     of the lowest-order one-bit in the specified value, or zero if
	 *     the specified value is itself equal to zero.
	 * @since 1.5
	 */
	public static long lowestOneBit(long i) {
		// HD, Section 2-1
		//Java中的负数使用补码的形式表示（反码 + 1）
		//反码和原码去于操作，肯定为0，但是补码，通过+1，
		//与操作则可以保存右边最小位不为0的位数
		return i & -i;
	}
	
	/**
	 * Returns the number of zero bits preceding the highest-order
	 * ("leftmost") one-bit in the two's complement binary representation
	 * of the specified {@code long} value.  Returns 64 if the
	 * specified value has no one-bits in its two's complement representation,
	 * in other words if it is equal to zero.
	 *
	 * 返回二进制中左边为0的位数,如返回64位则该值为0.
	 *
	 * <p>Note that this method is closely related to the logarithm base 2.
	 * For all positive {@code long} values x:
	 * <ul>
	 * <li>floor(log<sub>2</sub>(x)) = {@code 63 - numberOfLeadingZeros(x)}
	 * <li>ceil(log<sub>2</sub>(x)) = {@code 64 - numberOfLeadingZeros(x - 1)}
	 * </ul>
	 *
	 * 该方法和2的对数的计算有关, 对于x, log2(x) 向下取整: 31 - noz(x)
	 * 向上取整: 32 - noz(x - 1)
	 *
	 * @param i the value whose number of leading zeros is to be computed
	 * @return the number of zero bits preceding the highest-order
	 *     ("leftmost") one-bit in the two's complement binary representation
	 *     of the specified {@code long} value, or 64 if the value
	 *     is equal to zero.
	 * @since 1.5
	 */
	public static int numberOfLeadingZeros(long i) {
		// HD, Figure 5-6
		 if (i == 0)
			return 64;
		int n = 1;
		//按照对半的原则, 每次处理检查一半的位数,直到一位
		int x = (int)(i >>> 32); //切半
		if (x == 0) { n += 32; x = (int)i; } //如果前32位都为0,取后32位
		//依次切半,然后尾数补0,让有符号的数不断前移
		if (x >>> 16 == 0) { n += 16; x <<= 16; }
		if (x >>> 24 == 0) { n +=  8; x <<=  8; }
		if (x >>> 28 == 0) { n +=  4; x <<=  4; }
		if (x >>> 30 == 0) { n +=  2; x <<=  2; }
		n -= x >>> 31;
		return n;
	}
	
	/**
	 * Returns the number of zero bits following the lowest-order ("rightmost")
	 * one-bit in the two's complement binary representation of the specified
	 * {@code long} value.  Returns 64 if the specified value has no
	 * one-bits in its two's complement representation, in other words if it is
	 * equal to zero.
	 *
	 * 返回尾部的0的个数（连续为0直到end）
	 *
	 * @param i the value whose number of trailing zeros is to be computed
	 * @return the number of zero bits following the lowest-order ("rightmost")
	 *     one-bit in the two's complement binary representation of the
	 *     specified {@code long} value, or 64 if the value is equal
	 *     to zero.
	 * @since 1.5
	 */
	public static int numberOfTrailingZeros(long i) {
		// HD, Figure 5-14
		int x, y;
		if (i == 0) return 64;
		//按照对半的原则, 每次处理检查一半的位数,直到一位
		int n = 63;
		y = (int)i; if (y != 0) { n = n -32; x = y; } else x = (int)(i>>>32);
		y = x <<16; if (y != 0) { n = n -16; x = y; }
		y = x << 8; if (y != 0) { n = n - 8; x = y; }
		y = x << 4; if (y != 0) { n = n - 4; x = y; }
		y = x << 2; if (y != 0) { n = n - 2; x = y; }
		return n - ((x << 1) >>> 31);
	}
	
	/**
	 * Returns the number of one-bits in the two's complement binary
	 * representation of the specified {@code long} value.  This function is
	 * sometimes referred to as the <i>population count</i>.
	 *
	 * 返回bit位为1的个数 
	原理解析：以2位为标准,将2bit内存有多少位1存入该bit中,如01需要存储一位,存储成01.
	11有两位存储为10.我们就可以 得到00 = 00 - 00, 01 = 01 - 00/ 10 - 01, 10 = 11 - 01.
	存储的计算方法为: x = i - (i >>> 1). 但是如果位数不止2位呢, 就会出现冲突,如1110,
	应该存储为1001(前2个2位1,后两个1位1),应该减去0101,我们就不能简单的使用i>>>1,
	因为前面的移位 动作会影响到后面的位数. 移位之后变成0111,之前第三位应该为0的,
	被上一位抢占了位置.我们可以发现,我们2位中移位之后前面 都是补0操作,
	那我们可以手动把这些占据的位置给重新置为0, 0111 & 0101 = 0101,
	这里就将第三位被强占的位置重新置为了0. 所以在面对多位计算的时候,
	我们可以将占位符先置为0: & 0101 0101 即 & 0x55555555. 
	这里的第一步就是计算每两位中有几个1, 并存储到本地. 这时候, 存储当前情况应该如下: 
	AABB CCDD EEFF ...., 每两位存储该两位中1的个数. 我们需要计算4位bit中存 在几个1,
	即AA + BB, CC + DD, AABB & 0011 = 00BB,这样可以取到BB的值(同理CCDD & 0011 == DD), 
	(i >>> 2) & 0011 = 00AA, 可以取到AA的值(同理BBCC & 0011=CC(BB移动过来了,但是不影响CC的取值)). 
	即00BB 00DD + 00AA 00CC,这样前4个bit为就可以存储 AA + BB的值,即前4个bit位中1个数的值. 
	所以第二步就是计算每4位bit中1的个数并存储到该位置中.这时候的数据应该如下:  
	AAAA BBBB CCCC,这时候我们需要计算前8位中的bit值, 这时候我们进行移位即可, AAAA BBBB 
	CCCC DDDD + 0000 AAAA BBBBB CCCC 我们可以发现我们需要只是第二部分(BBBB + AAAA)和
	第四部分(DDDD + CCCC), 所以我们 &0f0f0f0f,去掉中间部分.所以第三步就是 
	存储前8位中1的个数并存储在8位bit中. 这时候的数据应该是: AAAA AAAA BBBB BBBB CCCC 
	CCCC DDDD DDDD,那我们计算同理计算前 16位的1的个数:0000 0000 AAAA AAAA BBBB BBBB 
	CCCC CCCC, 相加得到:(A)(A+B)(B+C)(C+D), 这里的第一部分是由信息丢失的,所以 
	第四步是计算前16位的和, 第五步计算32位,同理移位可得: (A)(A+B)(A+B+C)(A+B+C+D),
	同理中, 最后一步计算(A+B+C+D+E+F+G+H)的和.
	同样存在数据丢失,但我们只需要最后一个 部分的信息. long有64bit最多有64位，
	取最后7位bit位(2^7)存储的和即可, 所以&0x7f即可.
	 
	 * @param i the value whose bits are to be counted
	 * @return the number of one-bits in the two's complement binary
	 *     representation of the specified {@code long} value.
	 * @since 1.5
	 */
	 public static int bitCount(long i) {
		// HD, Figure 5-14
		//计算2位bit中1的个数,存储在2位中
		i = i - ((i >>> 1) & 0x5555555555555555L);
		//计算4为bit中的1的个数, 存储在4为bit中
		i = (i & 0x3333333333333333L) + ((i >>> 2) & 0x3333333333333333L);
		//计算8位
		i = (i + (i >>> 4)) & 0x0f0f0f0f0f0f0f0fL;
		//计算16位
		i = i + (i >>> 8);
		//计算32位
		i = i + (i >>> 16);
		//计算64位
		i = i + (i >>> 32);
		return (int)i & 0x7f;
	}
	 
	/**
	 * Returns the value obtained by rotating the two's complement binary
	 * representation of the specified {@code long} value left by the
	 * specified number of bits.  (Bits shifted out of the left hand, or
	 * high-order, side reenter on the right, or low-order.)
	 *
	 * 向左移动二进制数,超出范围的数会从右边重新进入.
	 *
	 * <p>Note that left rotation with a negative distance is equivalent to
	 * right rotation: {@code rotateLeft(val, -distance) == rotateRight(val,
	 * distance)}.  Note also that rotation by any multiple of 64 is a
	 * no-op, so all but the last six bits of the rotation distance can be
	 * ignored, even if the distance is negative: {@code rotateLeft(val,
	 * distance) == rotateLeft(val, distance & 0x3F)}.
	 *
	 * 注意移动64倍位是个空操作,没有影响, 并且推荐 distance & 0x3F,
	 * 倍数是个空操作, 只用取后6位bit为代表的距离. 就算是负数也是一样,只取后6位即可.
	 *
	 * @param i the value whose bits are to be rotated left
	 * @param distance the number of bit positions to rotate left
	 * @return the value obtained by rotating the two's complement binary
	 *     representation of the specified {@code long} value left by the
	 *     specified number of bits.
	 * @since 1.5
	 */
	public static long rotateLeft(long i, int distance) {
		//首先向左移动dis的距离,右边空出dis的位数被补0. ==> i << distance
		//无符号右移动64 - dis的距离进行, 取或填充0的位置. ==> i >>> -distance
		//为什么-distance为32 - dis. 负数移位同样取后6位, 负数为补码(反码+1),
		//原码(正数) + 补码(负数) = 原码 + 反码 + 1, 由于原码和反码对应位取反
		//肯定最后结果为11 1111, +1之后为100 0000为64. 负数 = 64 - dis.
		//如5的原码:00 0101, 补码为: 11 1011为(值为59)
		return (i << distance) | (i >>> -distance);
	}
	
	/**
	 * Returns the value obtained by rotating the two's complement binary
	 * representation of the specified {@code long} value right by the
	 * specified number of bits.  (Bits shifted out of the right hand, or
	 * low-order, side reenter on the left, or high-order.)
	 *
	 * 同理右移动,左进入 
	 *
	 * <p>Note that right rotation with a negative distance is equivalent to
	 * left rotation: {@code rotateRight(val, -distance) == rotateLeft(val,
	 * distance)}.  Note also that rotation by any multiple of 64 is a
	 * no-op, so all but the last six bits of the rotation distance can be
	 * ignored, even if the distance is negative: {@code rotateRight(val,
	 * distance) == rotateRight(val, distance & 0x3F)}.
	 *
	 * @param i the value whose bits are to be rotated right
	 * @param distance the number of bit positions to rotate right
	 * @return the value obtained by rotating the two's complement binary
	 *     representation of the specified {@code long} value right by the
	 *     specified number of bits.
	 * @since 1.5
	 */
	public static long rotateRight(long i, int distance) {
		return (i >>> distance) | (i << -distance);
	}
	
	/**
	 * Returns the value obtained by reversing the order of the bits in the
	 * two's complement binary representation of the specified {@code long}
	 * value.
	 *
	 * bit位的转置
	 *
	 * @param i the value to be reversed
	 * @return the value obtained by reversing order of the bits in the
	 *     specified {@code long} value.
	 * @since 1.5
	 */
	public static long reverse(long i) {
		// HD, Figure 7-1
		//以两位为基准，更换相邻的bit位,&0x555...(0101): 存储偶数位，清空
		//奇数位，然后移动到前面(<<1)。(i >>> 1) & 0x555...: 存储奇数位，移动
		//到后面,清空偶数位。做或操作结合在一起。由ABCD EFGH -> BADC FEHG
		i = (i & 0x5555555555555555L) << 1 | (i >>> 1) & 0x5555555555555555L;
		//以4位为标准交换相邻2位的位置，首先&0x333...(0011), 保留后两位，并
		//移动到前面.(i >>> 2) & 0x333...,保留前两位并移动到后面。
		//由BADC FEHG ==> DCBA HGFE
		i = (i & 0x3333333333333333L) << 2 | (i >>> 2) & 0x3333333333333333L;
		//同理，以8位为标准移动位置，DCBA HGFE ==> HGFE DCBA
		i = (i & 0x0f0f0f0f0f0f0f0fL) << 4 | (i >>> 4) & 0x0f0f0f0f0f0f0f0fL;
		//同理，以16位为标准移动位置，dcba hgfe ==> hgfe dcba
		i = (i & 0x00ff00ff00ff00ffL) << 8 | (i >>> 8) & 0x00ff00ff00ff00ffL;
		//i << 48，将最后16位bit启动到最前面
		//((i & 0xffff0000) << 16),取第三个16位bit并移动到第二位
		//((i >>> 16) & 0xffff0000L)取第二个16位bit并移动到第三位, 
		//(i >>> 48)将第一个16位移动到第四位
		// A B C D (16位为基准) ==> D C B A 
		i = (i << 48) | ((i & 0xffff0000L) << 16) |
			((i >>> 16) & 0xffff0000L) | (i >>> 48);
		return i;
	}
	
	/**
	 * Returns the signum function of the specified {@code long} value.  (The
	 * return value is -1 if the specified value is negative; 0 if the
	 * specified value is zero; and 1 if the specified value is positive.)
	 *
	 * 传递的值如果为0返回0,正数返回1,负数返回-1
	 *
	 * @param i the value whose signum is to be computed
	 * @return the signum function of the specified {@code long} value.
	 * @since 1.5
	 */
	public static int signum(long i) {
		// HD, Section 2-7
		//0 ==> 0 | 0 = 0; 正数: 0 | 1 = 1 ,负数: -1 | 0 = -1
		//注意第一个是有符号右移(正数补0,负数补1),第二个为无符号右移(补0)
		return (int) ((i >> 63) | (-i >>> 63));
	}

	/**
	 * Returns the value obtained by reversing the order of the bytes in the
	 * two's complement representation of the specified {@code long} value.
	 *
	 * 按照byte进行转置
	 *
	 * @param i the value whose bytes are to be reversed
	 * @return the value obtained by reversing the bytes in the specified
	 *     {@code long} value.
	 * @since 1.5
	 */
	public static long reverseBytes(long i) {
		//直接调用reverse的最后两步
		//以16位为基础交换相邻8位的byte位
		i = (i & 0x00ff00ff00ff00ffL) << 8 | (i >>> 8) & 0x00ff00ff00ff00ffL;
		//reverse最后一步,交换4个16位的bit
		return (i << 48) | ((i & 0xffff0000L) << 16) |
			((i >>> 16) & 0xffff0000L) | (i >>> 48);
	}

	/**
	 * Adds two {@code long} values together as per the + operator.
	 * 求和
	 * @param a the first operand
	 * @param b the second operand
	 * @return the sum of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static long sum(long a, long b) {
		return a + b;
	}

	/**
	 * Returns the greater of two {@code long} values
	 * as if by calling {@link Math#max(long, long) Math.max}.
	 *
	 * 求较大值
	 *
	 * @param a the first operand
	 * @param b the second operand
	 * @return the greater of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static long max(long a, long b) {
		return Math.max(a, b);
	}

	/**
	 * Returns the smaller of two {@code long} values
	 * as if by calling {@link Math#min(long, long) Math.min}.
	 *
	 * 求较小值
	 * 
	 * @param a the first operand
	 * @param b the second operand
	 * @return the smaller of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static long min(long a, long b) {
		return Math.min(a, b);
	}

	/** use serialVersionUID from JDK 1.0.2 for interoperability */
	@Native private static final long serialVersionUID = 4290774380558885855L;
}
```

