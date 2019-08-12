# Integer

```java
package java.lang;

import java.lang.annotation.Native;

/**
 * The {@code Integer} class wraps a value of the primitive type
 * {@code int} in an object. An object of type {@code Integer}
 * contains a single field whose type is {@code int}.
 *
 * 
 * <p>In addition, this class provides several methods for converting
 * an {@code int} to a {@code String} and a {@code String} to an
 * {@code int}, as well as other constants and methods useful when
 * dealing with an {@code int}.
 *
 * <p>Implementation note: The implementations of the "bit twiddling(转换)"
 * methods (such as {@link #highestOneBit(int) highestOneBit} and
 * {@link #numberOfTrailingZeros(int) numberOfTrailingZeros}) are
 * based on material from Henry S. Warren, Jr.'s <i>Hacker's
 * Delight</i>, (Addison Wesley, 2002).
 *
 * @author  Lee Boynton
 * @author  Arthur van Hoff
 * @author  Josh Bloch
 * @author  Joseph D. Darcy
 * @since JDK1.0
 */
public final class Integer extends Number implements Comparable<Integer> {
	/**
	 * A constant holding the minimum value an {@code int} can
	 * have, -2<sup>31</sup>.
	 *
	 * int的最小值为负2的31次方（Java使用补码来显示负数）
	 * 添加@Native注释说明该常量可能被本地方法引用
	 *
	 */
	@Native public static final int   MIN_VALUE = 0x80000000;  

	/**
	 * A constant holding the maximum value an {@code int} can
	 * have, 2<sup>31</sup>-1.
	 *
	 * int的最大值为2的31次方减1
	 */
	@Native public static final int   MAX_VALUE = 0x7fffffff;

	/**
	 * The {@code Class} instance representing the primitive type
	 * {@code int}.
	 *
	 * 原始类型int在JVM中定义的Class对象
	 *
	 * @since   JDK1.1
	 */
	@SuppressWarnings("unchecked")
	public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

	/**
	 * All possible chars for representing a number as a String
	 *
	 */
	final static char[] digits = {
		'0' , '1' , '2' , '3' , '4' , '5' ,
		'6' , '7' , '8' , '9' , 'a' , 'b' ,
		'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,
		'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,
		'o' , 'p' , 'q' , 'r' , 's' , 't' ,
		'u' , 'v' , 'w' , 'x' , 'y' , 'z'
	};

	/**
	 * Returns a string representation of the first argument in the
	 * radix specified by the second argument.
	 *
	 * <p>If the radix is smaller than {@code Character.MIN_RADIX}(现为2)
	 * or larger than {@code Character.MAX_RADIX}(现为36), then the radix
	 * {@code 10} is used instead.
	 *
	 * <p>If the first argument is negative, the first element of the
	 * result is the ASCII minus character {@code '-'}
	 * ({@code '\u005Cu002D'}). If the first argument is not
	 * negative, no sign character appears in the result.
	 *
	 * <p>The remaining characters of the result represent the magnitude(大小)
	 * of the first argument. If the magnitude is zero, it is
	 * represented by a single zero character {@code '0'}
	 * ({@code '\u005Cu0030'}); otherwise, the first character of
	 * the representation of the magnitude will not be the zero
	 * character.  The following ASCII characters are used as digits:
	 *
	 * 返回的string类型,第一位要么为负号(-), 要么为第一位非0的数字(正数). 
	 *
	 * 下面为进制中可能使用的字母：
	 * <blockquote>
	 *   {@code 0123456789abcdefghijklmnopqrstuvwxyz}
	 * </blockquote>
	 *
	 * 如果是N进制(非10进制)的数字,则采用上面的前N个字符,用做位数代表.
	 *
	 * These are {@code '\u005Cu0030'} through
	 * {@code '\u005Cu0039'} and {@code '\u005Cu0061'} through
	 * {@code '\u005Cu007A'}. If {@code radix} is
	 * <var>N</var>, then the first <var>N</var> of these characters
	 * are used as radix-<var>N</var> digits in the order shown. Thus,
	 * the digits for hexadecimal (radix 16) are
	 * {@code 0123456789abcdef}. If uppercase letters are
	 * desired, the {@link java.lang.String#toUpperCase()} method may
	 * be called on the result:
	 *
	 * <blockquote>
	 *  {@code Integer.toString(n, 16).toUpperCase()}
	 * </blockquote>
	 *
	 * 返回不同进制(radix)表示的int类型数字的string对象. 大写转换时只放大后10位
	 * 非数字的字母.
	 *
	 * @param   i       an integer to be converted to a string.
	 * @param   radix   the radix to use in the string representation.
	 * @return  a string representation of the argument in the specified radix.
	 * @see     java.lang.Character#MAX_RADIX
	 * @see     java.lang.Character#MIN_RADIX
	 */
	public static String toString(int i, int radix) {
		//如果进制小于2,或者大于36就使用默认的10进制.
		if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
			radix = 10;
		
		/* Use the faster version */
		if (radix == 10) {
			//使用快速版本,进行转换
			return toString(i);
		}
		//新建数组进行存储，最长为33字符(二进制的时候, 一位符号)
		char buf[] = new char[33];
		boolean negative = (i < 0);
		int charPos = 32;

		if (!negative) {
			i = -i;  //i取负值
		}
		
		//相当于i的绝对值要大于radix ==> (|i| > radix)
		while (i <= -radix) {
			//取最后一位数字,并获取对应的编码
			buf[charPos--] = digits[-(i % radix)];
			i = i / radix;
		}
		//存储第一位数字
		buf[charPos] = digits[-i];
		//存储符号位
		if (negative) {
			buf[--charPos] = '-';
		}

		return new String(buf, charPos, (33 - charPos));
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
	 * <p>If the magnitude(大小) is zero, it is represented by a single zero
	 * character {@code '0'} ({@code '\u005Cu0030'}); otherwise,
	 * the first character of the representation of the magnitude will
	 * not be the zero character.
	 *
	 * <p>The behavior of radixes and the characters used as digits
	 * are the same as {@link #toString(int, int) toString}.
	 *
	 * @param   i       an integer to be converted to an unsigned string.
	 * @param   radix   the radix to use in the string representation.
	 * @return  an unsigned string representation of the argument in the specified radix.
	 * @see     #toString(int, int)
	 * @since 1.8
	 */
	public static String toUnsignedString(int i, int radix) {
		//调用Long的toUnsedString方法, 无符号之后数值的大小上限 x 2, 所以需要转换
		//为Long
		return Long.toUnsignedString(toUnsignedLong(i), radix);
	}

	/**
	 * Returns a string representation of the integer argument as an
	 * unsigned integer in base&nbsp;16.
	 *
	 * 返回无符号的参数(i)的16进制string表达式
	 *
	 * <p>The unsigned integer value is the argument plus 2<sup>32</sup>
	 * if the argument is negative; otherwise, it is equal to the
	 * argument.  This value is converted to a string of ASCII digits
	 * in hexadecimal (base&nbsp;16) with no extra leading
	 * {@code 0}s.
	 *
	 * 负数的话相当于加上了2^32, 正数和0值保持不变.转换的string对象没有前置0.
	 *
	 * <p>The value of the argument can be recovered from the returned
	 * string {@code s} by calling {@link
	 * Integer#parseUnsignedInt(String, int)
	 * Integer.parseUnsignedInt(s, 16)}.
	 *
	 * 获取的值s,可以通过Integer.paraseUnsignedInt(2,16)来恢复原先的值.
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
	 * {@code '\u005Cu0039'} and {@code '\u005Cu0061'} through
	 * {@code '\u005Cu0066'}. If uppercase letters are
	 * desired, the {@link java.lang.String#toUpperCase()} method may
	 * be called on the result:
	 *
	 * <blockquote>
	 *  {@code Integer.toHexString(n).toUpperCase()}
	 * </blockquote>
	 *
	 * @param   i   an integer to be converted to a string.
	 * @return  the string representation of the unsigned integer value
	 *          represented by the argument in hexadecimal (base&nbsp;16).
	 * @see #parseUnsignedInt(String, int)
	 * @see #toUnsignedString(int, int)
	 * @since   JDK1.0.2
	 */
	public static String toHexString(int i) {
		return toUnsignedString0(i, 4);
	}

	/**
	 * Returns a string representation of the integer argument as an
	 * unsigned integer in base&nbsp;8.
	 *
	 * 返回无符号的八进制表示形式，方法类似上面方法
	 *
	 * <p>The unsigned integer value is the argument plus 2<sup>32</sup>
	 * if the argument is negative; otherwise, it is equal to the
	 * argument.  This value is converted to a string of ASCII digits
	 * in octal (base&nbsp;8) with no extra leading {@code 0}s.
	 *
	 * <p>The value of the argument can be recovered from the returned
	 * string {@code s} by calling {@link
	 * Integer#parseUnsignedInt(String, int)
	 * Integer.parseUnsignedInt(s, 8)}.
	 *
	 * <p>If the unsigned magnitude is zero, it is represented by a
	 * single zero character {@code '0'} ({@code '\u005Cu0030'});
	 * otherwise, the first character of the representation of the
	 * unsigned magnitude will not be the zero character. The
	 * following characters are used as octal digits:
	 *
	 * <blockquote>
	 * {@code 01234567}
	 * </blockquote>
	 *
	 * These are the characters {@code '\u005Cu0030'} through
	 * {@code '\u005Cu0037'}.
	 *
	 * @param   i   an integer to be converted to a string.
	 * @return  the string representation of the unsigned integer value
	 *          represented by the argument in octal (base&nbsp;8).
	 * @see #parseUnsignedInt(String, int)
	 * @see #toUnsignedString(int, int)
	 * @since   JDK1.0.2
	 */
	public static String toOctalString(int i) {
		return toUnsignedString0(i, 3);
	}

	/**
	 * Returns a string representation of the integer argument as an
	 * unsigned integer in base&nbsp;2.
	 *
	 * 返回无符号的二进制表示形式，方法类似上面方法
	 *
	 * <p>The unsigned integer value is the argument plus 2<sup>32</sup>
	 * if the argument is negative; otherwise it is equal to the
	 * argument.  This value is converted to a string of ASCII digits
	 * in binary (base&nbsp;2) with no extra leading {@code 0}s.
	 *
	 * <p>The value of the argument can be recovered from the returned
	 * string {@code s} by calling {@link
	 * Integer#parseUnsignedInt(String, int)
	 * Integer.parseUnsignedInt(s, 2)}.
	 *
	 * <p>If the unsigned magnitude is zero, it is represented by a
	 * single zero character {@code '0'} ({@code '\u005Cu0030'});
	 * otherwise, the first character of the representation of the
	 * unsigned magnitude will not be the zero character. The
	 * characters {@code '0'} ({@code '\u005Cu0030'}) and {@code
	 * '1'} ({@code '\u005Cu0031'}) are used as binary digits.
	 *
	 * @param   i   an integer to be converted to a string.
	 * @return  the string representation of the unsigned integer value
	 *          represented by the argument in binary (base&nbsp;2).
	 * @see #parseUnsignedInt(String, int)
	 * @see #toUnsignedString(int, int)
	 * @since   JDK1.0.2
	 */
	public static String toBinaryString(int i) {
		return toUnsignedString0(i, 1);
	}	

	/**
	 * Convert the integer to an unsigned number.
	 * 
	 * 转换integer类型为无符号数字
	 *
	 */
	private static String toUnsignedString0(int val, int shift) {
		// assert shift > 0 && shift <=5 : "Illegal shift value";
		//获取数字有效的位数(非0位)
		int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
		
		//计算所占的字符长度
		int chars = Math.max(((mag + (shift - 1)) / shift), 1);
		char[] buf = new char[chars];

		formatUnsignedInt(val, shift, buf, 0, chars);

		// Use special constructor which takes over "buf".
		return new String(buf, true);
	}

	/**
	 * 将int值转换为无符号的数字，并存储到对应的数组中
	 *
	 * Format a long (treated as unsigned) into a character buffer.
	 * @param val the unsigned int to format
	 * @param shift the log2 of the base to format in (4 for hex, 3 for octal, 1 for binary)
	 * @param buf the character buffer to write to
	 * @param offset the offset in the destination buffer to start at
	 * @param len the number of characters to write
	 * @return the lowest character  location used
	 */
	 static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
		int charPos = len;
		int radix = 1 << shift;
		int mask = radix - 1;
		do {
			//每次处理shift位二进制数， 直到为0
			buf[offset + --charPos] = Integer.digits[val & mask];
			val >>>= shift;
		} while (val != 0 && charPos > 0);

		return charPos;
	}

	/**
	 * Returns a {@code String} object representing the
	 * specified integer. The argument is converted to signed(带符号的) decimal
	 * representation and returned as a string, exactly as if the
	 * argument and radix 10 were given as arguments to the {@link
	 * #toString(int, int)} method.
	 *
	 * 返回10进制表示的string类型, 也是默认toString()调用的方法
	 *
	 * @param   i   an integer to be converted.
	 * @return  a string representation of the argument in base&nbsp;10.
	 */
	public static String toString(int i) {
		if (i == Integer.MIN_VALUE)
			return "-2147483648";
		//获取i的位数, 负数自动加一(留给符号位)
		int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i); 
		char[] buf = new char[size];
		//获取字符表示形式
		getChars(i, size, buf);
		return new String(buf, true);
	}

	final static char [] DigitTens = {
		'0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
		'1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
		'2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
		'3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
		'4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
		'5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
		'6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
		'7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
		'8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
		'9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
		} ;

	final static char [] DigitOnes = {
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
		} ;

	/**
	 * Returns a string representation of the argument as an unsigned
	 * decimal value.
	 *
	 * 返回int参数的无符号值(上限x2,int无法存储), 转换为long后调用long
	 * 的转换方法
	 *
	 * The argument is converted to unsigned decimal representation
	 * and returned as a string exactly as if the argument and radix
	 * 10 were given as arguments to the {@link #toUnsignedString(int,
	 * int)} method.
	 *
	 * @param   i  an integer to be converted to an unsigned string.
	 * @return  an unsigned string representation of the argument.
	 * @see     #toUnsignedString(int, int)
	 * @since 1.8
	 */
	public static String toUnsignedString(int i) {
		return Long.toString(toUnsignedLong(i));
	}
		

	/**
	 * Places characters representing the integer i into the
	 * character array buf. The characters are placed into
	 * the buffer backwards starting with the least significant
	 * digit at the specified index (exclusive), and working
	 * backwards from there.
	 *
	 * 将i的显示字符从位置大到小放入char[]数组中. 
	 * 如果i为整形的最小值,则会出现问题(对最小值取负还是原来的值,
	 * 在后面数组值读取时会越界)
	 *
	 * Will fail if i == Integer.MIN_VALUE
	 */
	static void getChars(int i, int index, char[] buf) {
		int q, r;
		int charPos = index;
		char sign = 0;

		if (i < 0) { //记录负数的符号位并转换数字
			sign = '-';
			i = -i;
		}

		// Generate two digits per iteration, 每次处理两位, 知道小于65536(2^16)
		while (i >= 65536) {
			q = i / 100;
		// really: r = i - (q * 100);
		//相当于 r = i - (q * 64 + q * 32 + q * 4) = i - q * 100
		//r取的值就为i的后两位(十位和个位)
			r = i - ((q << 6) + (q << 5) + (q << 2));
			i = q;
			buf [--charPos] = DigitOnes[r]; //存储个位
			buf [--charPos] = DigitTens[r]; //存储十位
		}

		// Fall thru to fast mode for smaller numbers
		// assert(i <= 65536, i);
		for (;;) {
			//相当于 q = i * (52429 / 2 ^ 19) = i * 0.1000003
			q = (i * 52429) >>> (16+3);
			//相当于 r = i - q * 10 ,取得值为个位数
			r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
			buf [--charPos] = digits [r];
			i = q;
			if (i == 0) break;
		}
		if (sign != 0) {
			//添加符号位
			buf [--charPos] = sign;
		}
	}

	final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
									  99999999, 999999999, Integer.MAX_VALUE };

	// Requires positive x
	static int stringSize(int x) {
		for (int i=0; ; i++)
			if (x <= sizeTable[i])
				return i+1;
	}

	/**
	 * Parses the string argument as a signed integer in the radix
	 * specified by the second argument. The characters in the string
	 * must all be digits of the specified radix (as determined by
	 * whether {@link java.lang.Character#digit(char, int)} returns a
	 * nonnegative value), except that the first character may be an
	 * ASCII minus sign {@code '-'} ({@code '\u005Cu002D'}) to
	 * indicate a negative value or an ASCII plus sign {@code '+'}
	 * ({@code '\u005Cu002B'}) to indicate a positive value. The
	 * resulting integer value is returned.
	 *
	 * 按照对应的进制（radix），将string转换为int.
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
	 * radix, except that the first character may be a minus sign
	 * {@code '-'} ({@code '\u005Cu002D'}) or plus sign
	 * {@code '+'} ({@code '\u005Cu002B'}) provided that the
	 * string is longer than length 1.
	 *
	 * <li>The value represented by the string is not a value of type
	 * {@code int}.
	 * </ul>
	 *
	 * <p>Examples:
	 * <blockquote><pre>
	 * parseInt("0", 10) returns 0
	 * parseInt("473", 10) returns 473
	 * parseInt("+42", 10) returns 42
	 * parseInt("-0", 10) returns 0
	 * parseInt("-FF", 16) returns -255
	 * parseInt("1100110", 2) returns 102
	 * parseInt("2147483647", 10) returns 2147483647
	 * parseInt("-2147483648", 10) returns -2147483648
	 * parseInt("2147483648", 10) throws a NumberFormatException
	 * parseInt("99", 8) throws a NumberFormatException
	 * parseInt("Kona", 10) throws a NumberFormatException
	 * parseInt("Kona", 27) returns 411787
	 * </pre></blockquote>
	 *
	 * @param      s   the {@code String} containing the integer
	 *                  representation to be parsed
	 * @param      radix   the radix to be used while parsing {@code s}.
	 * @return     the integer represented by the string argument in the
	 *             specified radix.
	 * @exception  NumberFormatException if the {@code String}
	 *             does not contain a parsable {@code int}.
	 */
	public static int parseInt(String s, int radix)
				throws NumberFormatException
	{
		/*
		 * WARNING: This method may be invoked early during VM initialization
		 * before IntegerCache is initialized. Care must be taken to not use
		 * the valueOf method.
		 */

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
		
		int result = 0;
		boolean negative = false;
		int i = 0, len = s.length();
		int limit = -Integer.MAX_VALUE;
		int multmin;
		int digit;

		if (len > 0) {
			char firstChar = s.charAt(0);
			if (firstChar < '0') { // Possible leading "+" or "-"
				if (firstChar == '-') {
					negative = true;
					limit = Integer.MIN_VALUE;
				} else if (firstChar != '+')
					throw NumberFormatException.forInputString(s);

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
				result *= radix; //result累乘
				if (result < limit + digit) { //判断加上digit之后是否超出了范围
					throw NumberFormatException.forInputString(s);
				}
				result -= digit;
			}
		} else {
			throw NumberFormatException.forInputString(s);
		}
		return negative ? result : -result;
	}

	/**
	 * Parses the string argument as a signed decimal integer. The
	 * characters in the string must all be decimal digits, except
	 * that the first character may be an ASCII minus sign {@code '-'}
	 * ({@code '\u005Cu002D'}) to indicate a negative value or an
	 * ASCII plus sign {@code '+'} ({@code '\u005Cu002B'}) to
	 * indicate a positive value. The resulting integer value is
	 * returned, exactly as if the argument and the radix 10 were
	 * given as arguments to the {@link #parseInt(java.lang.String,
	 * int)} method.
	 *
	 * 默认10进制，调用上面的方法进行转换
	 *
	 * @param s    a {@code String} containing the {@code int}
	 *             representation to be parsed
	 * @return     the integer value represented by the argument in decimal.
	 * @exception  NumberFormatException  if the string does not contain a
	 *               parsable integer.
	 */
	public static int parseInt(String s) throws NumberFormatException {
		return parseInt(s,10);
	}

	/**
	 * Parses the string argument as an unsigned integer in the radix
	 * specified by the second argument.  An unsigned integer maps the
	 * values usually associated with negative numbers to positive
	 * numbers larger than {@code MAX_VALUE}.
	 *
	 * 将一个string对象转换为一个无符号int类型
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
	 * largest unsigned {@code int}, 2<sup>32</sup>-1.
	 *
	 * int类型的无符号数字的最大值为： 2^32 - 1.(有符号： 2^31 - 1)
	 *
	 * </ul>
	 *
	 *
	 * @param      s   the {@code String} containing the unsigned integer
	 *                  representation to be parsed
	 * @param      radix   the radix to be used while parsing {@code s}.
	 * @return     the integer represented by the string argument in the
	 *             specified radix.
	 * @throws     NumberFormatException if the {@code String}
	 *             does not contain a parsable {@code int}.
	 * @since 1.8
	 */
	public static int parseUnsignedInt(String s, int radix)
				throws NumberFormatException {
		if (s == null)  {
			throw new NumberFormatException("null");
		}

		int len = s.length();
		if (len > 0) {
			char firstChar = s.charAt(0);
			if (firstChar == '-') {  //无符号不允许出现负数标识： -
				throw new
					NumberFormatException(String.format("Illegal leading minus sign " +
													   "on unsigned string %s.", s));
			} else {
				//如果String s的代表的数字大小不超过有符号的范围，直接调用有符号转换(int转换为10进制最多9位)
				if (len <= 5 || // Integer.MAX_VALUE in Character.MAX_RADIX is 6 digits 36进制有符号最多6位
					(radix == 10 && len <= 9) ) { // Integer.MAX_VALUE in base 10 is 10 digits 10进制最多9位
					return parseInt(s, radix);
				} else {
					long ell = Long.parseLong(s, radix); //调用Long的无符号转换功能，进行转换
					if ((ell & 0xffff_ffff_0000_0000L) == 0) { //进行位检查，是否超过了无符号int的范围
						return (int) ell;
					} else {
						throw new
							NumberFormatException(String.format("String value %s exceeds " +
																"range of unsigned int.", s));
					}
				}
			}
		} else {
			throw NumberFormatException.forInputString(s);
		}
	}

	/**
	 * Parses the string argument as an unsigned decimal integer. The
	 * characters in the string must all be decimal digits, except
	 * that the first character may be an an ASCII plus sign {@code
	 * '+'} ({@code '\u005Cu002B'}). The resulting integer value
	 * is returned, exactly as if the argument and the radix 10 were
	 * given as arguments to the {@link
	 * #parseUnsignedInt(java.lang.String, int)} method.
	 *
	 * 不传递radix，默认使用10
	 *
	 * @param s   a {@code String} containing the unsigned {@code int}
	 *            representation to be parsed
	 * @return    the unsigned integer value represented by the argument in decimal.
	 * @throws    NumberFormatException  if the string does not contain a
	 *            parsable unsigned integer.
	 * @since 1.8
	 */
	public static int parseUnsignedInt(String s) throws NumberFormatException {
		return parseUnsignedInt(s, 10);
	}

	/**
	 * Returns an {@code Integer} object holding the value
	 * extracted from the specified {@code String} when parsed
	 * with the radix given by the second argument. The first argument
	 * is interpreted(解读) as representing a signed integer in the radix
	 * specified by the second argument, exactly as if the arguments
	 * were given to the {@link #parseInt(java.lang.String, int)}
	 * method. The result is an {@code Integer} object that
	 * represents the integer value specified by the string.
	 *
	 * 调用parseInt(String s, int radix)进行转换，并获取对应的实例对象
	 *
	 * <p>In other words, this method returns an {@code Integer}
	 * object equal to the value of:
	 *
	 * <blockquote>
	 *  {@code new Integer(Integer.parseInt(s, radix))}
	 * </blockquote>
	 *
	 * @param      s   the string to be parsed.
	 * @param      radix the radix to be used in interpreting {@code s}
	 * @return     an {@code Integer} object holding the value
	 *             represented by the string argument in the specified
	 *             radix.
	 * @exception NumberFormatException if the {@code String}
	 *            does not contain a parsable {@code int}.
	 */
	public static Integer valueOf(String s, int radix) throws NumberFormatException {
		return Integer.valueOf(parseInt(s,radix));
	}

	/**
	 * Returns an {@code Integer} object holding the
	 * value of the specified {@code String}. The argument is
	 * interpreted as representing a signed decimal integer, exactly
	 * as if the argument were given to the {@link
	 * #parseInt(java.lang.String)} method. The result is an
	 * {@code Integer} object that represents the integer value
	 * specified by the string.
	 *
	 * 不传递radix，默认使用10
	 *
	 * <p>In other words, this method returns an {@code Integer}
	 * object equal to the value of:
	 *
	 * <blockquote>
	 *  {@code new Integer(Integer.parseInt(s))}
	 * </blockquote>
	 *
	 * @param      s   the string to be parsed.
	 * @return     an {@code Integer} object holding the value
	 *             represented by the string argument.
	 * @exception  NumberFormatException  if the string cannot be parsed
	 *             as an integer.
	 */
	public static Integer valueOf(String s) throws NumberFormatException {
		return Integer.valueOf(parseInt(s, 10));
	}

	/**
	 * Cache to support the object identity(身份/区分) semantics(语义) of autoboxing for values between
	 * -128 and 127 (inclusive) as required by JLS.
	 *
	 * The cache is initialized on first usage.  The size of the cache
	 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
	 * During VM initialization, java.lang.Integer.IntegerCache.high property
	 * may be set and saved in the private system properties in the
	 * sun.misc.VM class.
	 */

	private static class IntegerCache {
		static final int low = -128;
		static final int high;
		static final Integer cache[];

		static { //静态语句中，在类的初始化的时候进行缓存的初始化
			// high value may be configured by property
			int h = 127;
			String integerCacheHighPropValue =
				sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
			if (integerCacheHighPropValue != null) {
				try {
					int i = parseInt(integerCacheHighPropValue);
					i = Math.max(i, 127);
					// Maximum array size is Integer.MAX_VALUE
					h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
				} catch( NumberFormatException nfe) {
					// If the property cannot be parsed into an int, ignore it.
				}
			}
			high = h;

			cache = new Integer[(high - low) + 1];
			int j = low;
			for(int k = 0; k < cache.length; k++)
				cache[k] = new Integer(j++);

			// range [-128, 127] must be interned (JLS7 5.1.7)
			assert IntegerCache.high >= 127;
		}

		private IntegerCache() {} //私有构造函数，防止被外部访问
	}

	/**
	 * Returns an {@code Integer} instance representing the specified
	 * {@code int} value.  If a new {@code Integer} instance is not
	 * required, this method should generally be used in preference to
	 * the constructor {@link #Integer(int)}, as this method is likely
	 * to yield significantly better space and time performance by
	 * caching frequently requested values.
	 *
	 * 比直接调用构造函数有更好的性能和空间利用率（使用了缓存）
	 *
	 * This method will always cache values in the range -128 to 127,
	 * inclusive, and may cache other values outside of this range.
	 *
	 * 默认情况下缓存-128到127范围内的实例对象（返回缓存对象），取决于
	 * JVM是否设置了最大缓存范围。
	 *
	 * @param  i an {@code int} value.
	 * @return an {@code Integer} instance representing {@code i}.
	 * @since  1.5
	 */
	public static Integer valueOf(int i) {
		if (i >= IntegerCache.low && i <= IntegerCache.high)
			return IntegerCache.cache[i + (-IntegerCache.low)];
		return new Integer(i);
	}

	/**
	 * The value of the {@code Integer}.
	 *
	 * @serial
	 */
	private final int value;

	/**
	 * Constructs a newly allocated {@code Integer} object that
	 * represents the specified {@code int} value.
	 *
	 * @param   value   the value to be represented by the
	 *                  {@code Integer} object.
	 */
	public Integer(int value) {
		this.value = value;
	}

	/**
	 * Constructs a newly allocated {@code Integer} object that
	 * represents the {@code int} value indicated by the
	 * {@code String} parameter. The string is converted to an
	 * {@code int} value in exactly the manner used by the
	 * {@code parseInt} method for radix 10.
	 *
	 * @param      s   the {@code String} to be converted to an
	 *                 {@code Integer}.
	 * @exception  NumberFormatException  if the {@code String} does not
	 *               contain a parsable integer.
	 * @see        java.lang.Integer#parseInt(java.lang.String, int)
	 */
	public Integer(String s) throws NumberFormatException {
		this.value = parseInt(s, 10);
	}

	/**
	 * Returns the value of this {@code Integer} as a {@code byte}
	 * after a narrowing primitive conversion.
	 *
	 * 转换为byte值，取后8位字节码值
	 *
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public byte byteValue() {
		return (byte)value;
	}

	/**
	 * Returns the value of this {@code Integer} as a {@code short}
	 * after a narrowing primitive conversion.
	 *
	 * 转换为short值，取后16位字节码值
	 *
	 * @jls 5.1.3 Narrowing Primitive Conversions
	 */
	public short shortValue() {
		return (short)value;
	}

	/**
	 * Returns the value of this {@code Integer} as an
	 * {@code int}.
	 *
	 * 返回int值
	 */
	public int intValue() {
		return value;
	}

	/**
	 * Returns the value of this {@code Integer} as a {@code long}
	 * after a widening primitive conversion.
	 *
	 * int转换为long，向上转型，不会丢失精度。负数补1，正数补0。
	 *
	 * @jls 5.1.2 Widening Primitive Conversions
	 * @see Integer#toUnsignedLong(int)
	 */
	public long longValue() {
		return (long)value;
	}

	/**
	 * Returns the value of this {@code Integer} as a {@code float}
	 * after a widening primitive conversion.
	 *
	 * int转换为float，向上转型不会丢失精度
	 *
	 * @jls 5.1.2 Widening Primitive Conversions
	 */
	public float floatValue() {
		return (float)value;
	}

	/**
	 * Returns the value of this {@code Integer} as a {@code double}
	 * after a widening primitive conversion.
	 *
	 * int转换为double，向上转型不会丢失精度
	 *
	 * @jls 5.1.2 Widening Primitive Conversions
	 */
	public double doubleValue() {
		return (double)value;
	}

	/**
	 * Returns a {@code String} object representing this
	 * {@code Integer}'s value. The value is converted to signed
	 * decimal representation and returned as a string, exactly as if
	 * the integer value were given as an argument to the {@link
	 * java.lang.Integer#toString(int)} method.
	 *
	 * 将int转换为String对象，调用toString(value)方法,默认使用10进制
	 *
	 * @return  a string representation of the value of this object in
	 *          base&nbsp;10.
	 */
	public String toString() {
		return toString(value);
	}

	/**
	 * Returns a hash code for this {@code Integer}.
	 *
	 * 返回哈希码
	 *
	 * @return  a hash code value for this object, equal to the
	 *          primitive {@code int} value represented by this
	 *          {@code Integer} object.
	 */
	@Override
	public int hashCode() {
		return Integer.hashCode(value);
	}

	/**
	 * Returns a hash code for a {@code int} value; compatible with
	 * {@code Integer.hashCode()}.
	 *
	 * Integer的哈希码就是本身的value
	 *
	 * @param value the value to hash
	 * @since 1.8
	 *
	 * @return a hash code value for a {@code int} value.
	 */
	public static int hashCode(int value) {
		return value;
	}

	/**
	 * Compares this object to the specified object.  The result is
	 * {@code true} if and only if the argument is not
	 * {@code null} and is an {@code Integer} object that
	 * contains the same {@code int} value as this object.
	 *
	 * 覆盖Object的equals方法，比较内部的value值，不比较地址
	 *
	 * @param   obj   the object to compare with.
	 * @return  {@code true} if the objects are the same;
	 *          {@code false} otherwise.
	 */
	public boolean equals(Object obj) {
		if (obj instanceof Integer) {
			return value == ((Integer)obj).intValue();
		}
		return false;
	}

	/**
	 * Determines the integer value of the system property with the
	 * specified name.
	 *
	 * 获取系统指定属性（nm），所代表的Integer值
	 *
	 * <p>The first argument is treated as the name of a system
	 * property.  System properties are accessible through the {@link
	 * java.lang.System#getProperty(java.lang.String)} method. The
	 * string value of this property is then interpreted（解读） as an integer
	 * value using the grammar（语法） supported by {@link Integer#decode decode} and
	 * an {@code Integer} object representing this value is returned.
	 *
	 * 通过调用System.getProperty(nm)获取属性，并使用Integer.decode()进行解码
	 *
	 * <p>If there is no property with the specified name, if the
	 * specified name is empty or {@code null}, or if the property
	 * does not have the correct numeric format, then {@code null} is
	 * returned.
	 *
	 * 如果未找到对象或为空，返回null
	 *
	 * <p>In other words, this method returns an {@code Integer}
	 * object equal to the value of:
	 *
	 * <blockquote>
	 *  {@code getInteger(nm, null)}
	 * </blockquote>
	 *
	 * @param   nm   property name.
	 * @return  the {@code Integer} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     java.lang.System#getProperty(java.lang.String)
	 * @see     java.lang.System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Integer getInteger(String nm) {
		return getInteger(nm, null);
	}	

	/**
	 * Determines the integer value of the system property with the
	 * specified name.
	 *
	 * 如果没有找到对象或者为null,则返回第二个传递的参数值。
	 *
	 * <p>The first argument is treated as the name of a system
	 * property.  System properties are accessible through the {@link
	 * java.lang.System#getProperty(java.lang.String)} method. The
	 * string value of this property is then interpreted as an integer
	 * value using the grammar supported by {@link Integer#decode decode} and
	 * an {@code Integer} object representing this value is returned.
	 *
	 * <p>The second argument is the default value. An {@code Integer} object
	 * that represents the value of the second argument is returned if there
	 * is no property of the specified name, if the property does not have
	 * the correct numeric format, or if the specified name is empty or
	 * {@code null}.
	 *
	 * <p>In other words, this method returns an {@code Integer} object
	 * equal to the value of:
	 *
	 * <blockquote>
	 *  {@code getInteger(nm, new Integer(val))}
	 * </blockquote>
	 *
	 * but in practice it may be implemented in a manner such as:
	 *
	 * <blockquote><pre>
	 * Integer result = getInteger(nm, null);
	 * return (result == null) ? new Integer(val) : result;
	 * </pre></blockquote>
	 *
	 * to avoid the unnecessary allocation of an {@code Integer}
	 * object when the default value is not needed.
	 *
	 * @param   nm   property name.
	 * @param   val   default value.
	 * @return  the {@code Integer} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     java.lang.System#getProperty(java.lang.String)
	 * @see     java.lang.System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Integer getInteger(String nm, int val) {
		Integer result = getInteger(nm, null);
		return (result == null) ? Integer.valueOf(val) : result;
	}

	/**
	 * 同上，具体实现方法 
	 *
	 * Returns the integer value of the system property with the
	 * specified name.  The first argument is treated as the name of a
	 * system property.  System properties are accessible through the
	 * {@link java.lang.System#getProperty(java.lang.String)} method.
	 * The string value of this property is then interpreted as an
	 * integer value, as per the {@link Integer#decode decode} method,
	 * and an {@code Integer} object representing this value is
	 * returned; in summary:
	 *
	 * <ul><li>If the property value begins with the two ASCII characters
	 *         {@code 0x} or the ASCII character {@code #}, not
	 *      followed by a minus sign, then the rest of it is parsed as a
	 *      hexadecimal integer exactly as by the method
	 *      {@link #valueOf(java.lang.String, int)} with radix 16.
	 * <li>If the property value begins with the ASCII character
	 *     {@code 0} followed by another character, it is parsed as an
	 *     octal integer exactly as by the method
	 *     {@link #valueOf(java.lang.String, int)} with radix 8.
	 * <li>Otherwise, the property value is parsed as a decimal integer
	 * exactly as by the method {@link #valueOf(java.lang.String, int)}
	 * with radix 10.
	 * </ul>
	 *
	 * <p>The second argument is the default value. The default value is
	 * returned if there is no property of the specified name, if the
	 * property does not have the correct numeric format, or if the
	 * specified name is empty or {@code null}.
	 *
	 * @param   nm   property name.
	 * @param   val   default value.
	 * @return  the {@code Integer} value of the property.
	 * @throws  SecurityException for the same reasons as
	 *          {@link System#getProperty(String) System.getProperty}
	 * @see     System#getProperty(java.lang.String)
	 * @see     System#getProperty(java.lang.String, java.lang.String)
	 */
	public static Integer getInteger(String nm, Integer val) {
		String v = null;
		try {
			v = System.getProperty(nm);
		} catch (IllegalArgumentException | NullPointerException e) {
		}
		if (v != null) {
			try {
				return Integer.decode(v);
			} catch (NumberFormatException e) {
			}
		}
		return val;
	}	

	/**
	 * 解密String对象为Integer对象，接收十进制，十六进制，八进制
	 *
	 * Decodes a {@code String} into an {@code Integer}.
	 * Accepts decimal, hexadecimal, and octal numbers given
	 * by the following grammar:
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
	 * Integer.parseInt} method with the indicated(指示的) radix (10, 16, or
	 * 8).  This sequence of characters must represent a positive
	 * value or a {@link NumberFormatException} will be thrown.  The
	 * result is negated if first character of the specified {@code
	 * String} is the minus sign.  No whitespace characters are
	 * permitted in the {@code String}.
	 *
	 * @param     nm the {@code String} to decode.
	 * @return    an {@code Integer} object holding the {@code int}
	 *             value represented by {@code nm}
	 * @exception NumberFormatException  if the {@code String} does not
	 *            contain a parsable integer.
	 * @see java.lang.Integer#parseInt(java.lang.String, int)
	 */
	public static Integer decode(String nm) throws NumberFormatException {
		int radix = 10;
		int index = 0;
		boolean negative = false;
		Integer result;

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

		try {
			//调用Integer.valueOf()，进行转换
			result = Integer.valueOf(nm.substring(index), radix);
			result = negative ? Integer.valueOf(-result.intValue()) : result;
		} catch (NumberFormatException e) {
			// If number is Integer.MIN_VALUE, we'll end up here. The next line
			// handles this case, and causes any genuine format error to be
			// rethrown.
			String constant = negative ? ("-" + nm.substring(index))
									   : nm.substring(index);
			result = Integer.valueOf(constant, radix);
		}
		return result;
	}

	/**
	 *
	 * 实现Comparable接口中的比较方法，第一个参数小返回-1，等于0，大于1
	 *
	 * Compares two {@code Integer} objects numerically.
	 *
	 * @param   anotherInteger   the {@code Integer} to be compared.
	 * @return  the value {@code 0} if this {@code Integer} is
	 *          equal to the argument {@code Integer}; a value less than
	 *          {@code 0} if this {@code Integer} is numerically less
	 *          than the argument {@code Integer}; and a value greater
	 *          than {@code 0} if this {@code Integer} is numerically
	 *           greater than the argument {@code Integer} (signed
	 *           comparison).
	 * @since   1.2
	 */
	public int compareTo(Integer anotherInteger) {
		return compare(this.value, anotherInteger.value);
	}

	/**
	 * Compares two {@code int} values numerically.
	 * The value returned is identical to what would be returned by:
	 * <pre>
	 *    Integer.valueOf(x).compareTo(Integer.valueOf(y))
	 * </pre>
	 *
	 * @param  x the first {@code int} to compare
	 * @param  y the second {@code int} to compare
	 * @return the value {@code 0} if {@code x == y};
	 *         a value less than {@code 0} if {@code x < y}; and
	 *         a value greater than {@code 0} if {@code x > y}
	 * @since 1.7
	 */
	public static int compare(int x, int y) {
		return (x < y) ? -1 : ((x == y) ? 0 : 1);
	}

	/**
	 * Compares two {@code int} values numerically treating the values
	 * as unsigned.
	 *
	 * 比较值的无符号值大小
	 *
	 * @param  x the first {@code int} to compare
	 * @param  y the second {@code int} to compare
	 * @return the value {@code 0} if {@code x == y}; a value less
	 *         than {@code 0} if {@code x < y} as unsigned values; and
	 *         a value greater than {@code 0} if {@code x > y} as
	 *         unsigned values
	 * @since 1.8
	 */
	public static int compareUnsigned(int x, int y) {
		//在无符号int值中，上限是x2的，通过加上MIN_VALUE进行比较
		return compare(x + MIN_VALUE, y + MIN_VALUE);
	}

	/**
	 * Converts the argument to a {@code long} by an unsigned
	 * conversion.  In an unsigned conversion to a {@code long}, the
	 * high-order 32 bits of the {@code long} are zero and the
	 * low-order 32 bits are equal to the bits of the integer
	 * argument.
	 *
	 * 转换为long之后,前32位置0, 后32位保持为int的大小类型
	 *
	 * Consequently, zero and positive {@code int} values are mapped
	 * to a numerically equal {@code long} value and negative {@code
	 * int} values are mapped to a {@code long} value equal to the
	 * input plus 2<sup>32</sup>.
	 *
	 * 转换之后, 正数和0保持不变, 负数相当于原来的值加上2^32
	 *
	 * @param  x the value to convert to an unsigned {@code long}
	 * @return the argument converted to {@code long} by an unsigned
	 *         conversion
	 * @since 1.8
	 */
	public static long toUnsignedLong(int x) {
		return ((long) x) & 0xffffffffL;
	}

	/**
	 * Returns the unsigned quotient(商) of dividing the first argument by
	 * the second where each argument and the result is interpreted as
	 * an unsigned value.
	 *
	 * 无符号除法，通过转换为无符号long型进行计算
	 *
	 * <p>Note that in two's complement(补码) arithmetic(算术), the three other
	 * basic arithmetic operations of add, subtract, and multiply are
	 * bit-wise identical if the two operands are regarded as both
	 * being signed or both being unsigned. Therefore(因此) separate {@code
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
	public static int divideUnsigned(int dividend, int divisor) {
		// In lieu of tricky code, for now just use long arithmetic.
		return (int)(toUnsignedLong(dividend) / toUnsignedLong(divisor));
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
	public static int remainderUnsigned(int dividend, int divisor) {
		// In lieu of tricky code, for now just use long arithmetic.
		return (int)(toUnsignedLong(dividend) % toUnsignedLong(divisor));
	}

	// Bit twiddling (位数的切换)

	/**
	 * The number of bits used to represent an {@code int} value in two's
	 * complement(补充) binary form(补码的形式). 使用的二进制位数
	 *
	 * @since 1.5
	 */
	@Native public static final int SIZE = 32;

	/**
	 * The number of bytes used to represent a {@code int} value in two's
	 * complement(补充) binary form(补码的形式). 使用的byte数
	 *
	 * @since 1.8
	 */
	public static final int BYTES = SIZE / Byte.SIZE;

	/**
	 * Returns an {@code int} value with at most a single one-bit, in the
	 * position of the highest-order ("leftmost") one-bit in the specified
	 * {@code int} value.  Returns zero if the specified value has no
	 * one-bits in its two's complement binary representation, that is, if it
	 * is equal to zero.
	 *
	 * 返回左边最高位bit位所代表的值
	 *
	 * @param i the value whose highest one bit is to be computed
	 * @return an {@code int} value with a single one-bit, in the position
	 *     of the highest-order one-bit in the specified value, or zero if
	 *     the specified value is itself equal to zero.
	 * @since 1.5
	 */
	public static int highestOneBit(int i) {
		// HD, Figure 3-1
		i |= (i >>  1); //填充最高位右边一位为1，这是填充了1位
		i |= (i >>  2); //填充最高位右边两位为1，这时填充了3位
		i |= (i >>  4); //填充最高位右边四位为1，这时填充了7位
		i |= (i >>  8); //填充最高位右边八位为1，这时填充了15位
		i |= (i >> 16); //填充最高位右边十六位为1，这时填充了31位
		//现在保证了保证了i最高位后面全为1,减法之后保证i后面全为0
		return i - (i >>> 1);
	}

	/**
	 * Returns an {@code int} value with at most a single one-bit, in the
	 * position of the lowest-order ("rightmost") one-bit in the specified
	 * {@code int} value.  Returns zero if the specified value has no
	 * one-bits in its two's complement binary representation, that is, if it
	 * is equal to zero.
	 *
	 * 返回左边最低位bit位所代表的值
	 *
	 * @param i the value whose lowest one bit is to be computed
	 * @return an {@code int} value with a single one-bit, in the position
	 *     of the lowest-order one-bit in the specified value, or zero if
	 *     the specified value is itself equal to zero.
	 * @since 1.5
	 */
	public static int lowestOneBit(int i) {
		// HD, Section 2-1
		//Java中的负数使用补码的形式表示（反码 + 1）
		//反码和原码去于操作，肯定为0，但是补码，通过+1，
		//与操作则可以保存右边最小位不为0的位数
		return i & -i;
	}

	/**
	 * Returns the number of zero bits preceding the highest-order
	 * ("leftmost") one-bit in the two's complement binary representation
	 * of the specified {@code int} value.  Returns 32 if the
	 * specified value has no one-bits in its two's complement representation,
	 * in other words if it is equal to zero.
	 *
	 * 返回二进制中左边为0的位数,如返回32位则该值为0.
	 *
	 * <p>Note that this method is closely related to the logarithm(对数) base 2.
	 * For all positive {@code int} values x:
	 * <ul>
	 * <li>floor(log<sub>2</sub>(x)) = {@code 31 - numberOfLeadingZeros(x)}
	 * <li>ceil(log<sub>2</sub>(x)) = {@code 32 - numberOfLeadingZeros(x - 1)}
	 * </ul>
	 * 
	 * 该方法和2的对数的计算有关, 对于x, log2(x) 向下取整: 31 - noz(x)
	 * 向上取整: 32 - noz(x - 1)
	 *
	 * @param i the value whose number of leading zeros is to be computed
	 * @return the number of zero bits preceding the highest-order
	 *     ("leftmost") one-bit in the two's complement binary representation
	 *     of the specified {@code int} value, or 32 if the value
	 *     is equal to zero.
	 * @since 1.5
	 */
	public static int numberOfLeadingZeros(int i) {
		// HD, Figure 5-6
		if (i == 0)
			return 32;
		int n = 1;
		//按照对半的原则, 每次处理检查一半的位数,直到一位
		if (i >>> 16 == 0) { n += 16; i <<= 16; }
		if (i >>> 24 == 0) { n +=  8; i <<=  8; }
		if (i >>> 28 == 0) { n +=  4; i <<=  4; }
		if (i >>> 30 == 0) { n +=  2; i <<=  2; }
		n -= i >>> 31;
		return n;
	}

	/**
	 * Returns the number of zero bits following the lowest-order ("rightmost")
	 * one-bit in the two's complement binary representation of the specified
	 * {@code int} value.  Returns 32 if the specified value has no
	 * one-bits in its two's complement representation, in other words if it is
	 * equal to zero.
	 *
	 * 返回尾部的0的个数（连续为0直到end）
	 *
	 * @param i the value whose number of trailing zeros is to be computed
	 * @return the number of zero bits following the lowest-order ("rightmost")
	 *     one-bit in the two's complement binary representation of the
	 *     specified {@code int} value, or 32 if the value is equal
	 *     to zero.
	 * @since 1.5
	 */
	public static int numberOfTrailingZeros(int i) {
		// HD, Figure 5-14
		//按照对半的原则, 每次处理检查一半的位数,直到一位
		int y;
		if (i == 0) return 32;
		int n = 31;
		y = i <<16; if (y != 0) { n = n -16; i = y; }
		y = i << 8; if (y != 0) { n = n - 8; i = y; }
		y = i << 4; if (y != 0) { n = n - 4; i = y; }
		y = i << 2; if (y != 0) { n = n - 2; i = y; }
		return n - ((i << 1) >>> 31); //检查最后一位是否为0
	}

	/**
	 * Returns the number of one-bits in the two's complement binary
	 * representation of the specified {@code int} value.  This function is
	 * sometimes referred to as the <i>population count</i>.
	 *
	 * 返回bit位为1的个数 
	 *
	 * @param i the value whose bits are to be counted
	 * @return the number of one-bits in the two's complement binary
	 *     representation of the specified {@code int} value.
	 * @since 1.5
	 */
	 /*
	原理解析： 设 i = b31 x 2^31 + b30 x 2^30 + ... + b1 x 2^1 + b0 x 2^0
	我们最终需要达成的目标是：b31 + b30 + ... + b1 + b0
	第一步：i = i - ((i >>> 1) & 0x55555555) ===> 提取2位数
	i >>> 1 = 0 x 2^31 + b31 x 2^30 + b30 x 2^29 + ... + b2 x 2^1 + b1 x 2^0
	0x55555555: 0101 0101 0101 0101 0101 0101 0101 0101
	(i >>> 1) & 0x55555555 = b31 x 2^30 + b29 x 2^28 + ...+ b3 x 2^2 + b1 x 2^0
	i - ((i >>> 1) & 0x55555555) =
		b31 x 2^31 + (b30 - b31) x 2^30 + b29 x 2^29 + (b28 - b29) x 2^28 + ...
		+ b3 x 2^3 + (b2 -b3) x 2^2 + b1  x 2^1  + (b0 - b1) x 2^0
	   = (b31 + b30) x 2^30 + (b29 + b28) x 2^28 + ... + (b3 + b2) x 2^2 + (b1 + b0) x 2^0
	第二步：i = (i & 0x33333333) + ((i >>> 2) & 0x33333333); ==>提取4位
	i = (b31 + b30) x 2^30 + (b29 + b28) x 2^28 + ... + (b3 + b2) x 2^2 + (b1 + b0) x 2^0
	0x33333333 = 0011 0011 0011 0011 0011 0011 0011 0011
	i & 0x33333333 = (b29 + b28) x 2^28 + (b29 + b28) x 2^24 + ... + (b5 + b4) x 2^4 + (b1 + b0) x 2^0
	i >>> 2 = (b31 + b30) x 2^28 + (b29 + b28) x 2^26 + ... + (b5 + b4) x 2^2 + (b3 + b2) x 2^0
	i >>> 2 & 0x33333333 =
		(b31 + b30) x 2^28 + (b27 + b26) x 2^24 + ... + (b7 + b6) x 2^4 + (b3 + b2) x 2^0
	相加 = (b31 + b30 + b29 + b28) x 2^28 + ... + (b3 + b2 + b1 + b0) x 2^0
	第三步：i = (i + (i >>> 4)) & 0x0f0f0f0f =     ===>提取8位
		(b31 + b30 +...+ b25 + b24) x 2^24 + (b23 + b22 +...+ b17 + b16) x 2^16 +...+(b7 + b6 +...+b0) x 2^0
	第四步：i = i + (i >>> 8) =          ===>提取16位
		(b31 + b30 + ... + b24) x 2^24 +...+ (b15 + b14 +...+b0) x 2^0
	第五步: i = i + (i >>> 16);         ===>提取32位
		(b31 + b30 + ... + b24) x 2^24 +...+ (b31 + b30 +...+ b1 + b0) x 2^0
	最后一步：
	i = i + (i >>> 16); int有32bit最多有32位，取最后5位bit位(2^5)存储的和即可。
	
	原理解析2： 以2位为标准,将2bit内存有多少位1存入该bit中,如01需要存储一位,存储成01.
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
	第四步是计算前16位的和, 最后一步计算32位,同理移位可得: (A)(A+B)(A+B+C)(A+B+C+D),
	同样存在数据丢失,但我们只需要最后一个 部分的信息. int有32bit最多有32位，
	取最后5位bit位(2^5)存储的和即可, 所以&0x3f即可.
	 */
	public static int bitCount(int i) {
		// HD, Figure 5-2
		i = i - ((i >>> 1) & 0x55555555);
		i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
		i = (i + (i >>> 4)) & 0x0f0f0f0f;
		i = i + (i >>> 8);
		i = i + (i >>> 16);
		return i & 0x3f;
	}

	/**
	 * Returns the value obtained by rotating(旋转) the two's complement binary
	 * representation of the specified {@code int} value left by the
	 * specified number of bits.  (Bits shifted(移动) out of the left hand, or
	 * high-order, side reenter(重新进入) on the right, or low-order.)
	 *
	 * 向左移动二进制数,超出范围的数会从右边重新进入.
	 *
	 * <p>Note that left rotation with a negative distance is equivalent to
	 * right rotation: {@code rotateLeft(val, -distance) == rotateRight(val,
	 * distance)}.  Note also that rotation by any multiple of 32 is a
	 * no-op(空操作), so all but the last five bits of the rotation distance can be
	 * ignored, even if the distance is negative: {@code rotateLeft(val,
	 * distance) == rotateLeft(val, distance & 0x1F)}.
	 *
	 * 注意移动32倍位是个空操作,没有影响, 并且推荐 distance & 0x1F,前面的距离都是32的
	 * 倍数是个空操作, 只用取后5位bit为代表的距离. 就算是负数也是一样,只取后5位即可.
	 *
	 * @param i the value whose bits are to be rotated left
	 * @param distance the number of bit positions to rotate left
	 * @return the value obtained by rotating the two's complement binary
	 *     representation of the specified {@code int} value left by the
	 *     specified number of bits.
	 * @since 1.5
	 */
	public static int rotateLeft(int i, int distance) {
		//首先向左移动dis的距离,右边空出dis的位数被补0. ==> i << distance
		//无符号右移动32 - dis的距离进行, 取或填充0的位置. ==> i >>> -distance
		//为什么-distance为32 - dis. 负数移位同样取后5位, 负数为补码(反码+1),
		//原码(正数) + 补码(负数) = 原码 + 反码 + 1, 由于原码和反码对应位取反
		//肯定最后结果为1 1111, +1之后为10 0000为32. 负数 = 32 - dis.
		//如5的原码:0 0101, 补码为: 1 1011(值为27)
		return (i << distance) | (i >>> -distance);
	}

	/**
	 * Returns the value obtained by rotating the two's complement binary
	 * representation of the specified {@code int} value right by the
	 * specified number of bits.  (Bits shifted out of the right hand, or
	 * low-order, side reenter on the left, or high-order.)
	 *
	 * 同理右移动,左进入 
	 *
	 * <p>Note that right rotation with a negative distance is equivalent to
	 * left rotation: {@code rotateRight(val, -distance) == rotateLeft(val,
	 * distance)}.  Note also that rotation by any multiple of 32 is a
	 * no-op, so all but the last five bits of the rotation distance can be
	 * ignored, even if the distance is negative: {@code rotateRight(val,
	 * distance) == rotateRight(val, distance & 0x1F)}.
	 *
	 * @param i the value whose bits are to be rotated right
	 * @param distance the number of bit positions to rotate right
	 * @return the value obtained by rotating the two's complement binary
	 *     representation of the specified {@code int} value right by the
	 *     specified number of bits.
	 * @since 1.5
	 */
	public static int rotateRight(int i, int distance) {
		return (i >>> distance) | (i << -distance);
	}

	/**
	 * Returns the value obtained by reversing the order of the bits in the
	 * two's complement binary representation of the specified {@code int}
	 * value.
	 *
	 * bit位的转置
	 *
	 * @param i the value to be reversed
	 * @return the value obtained by reversing order of the bits in the
	 *     specified {@code int} value.
	 * @since 1.5
	 */
	public static int reverse(int i) {
		// HD, Figure 7-1
		//以两位为基准，更换相邻的bit位,&0x55555555(0101): 存储偶数位，清空
		//奇数位，然后移动到前面(<<1)。(i >>> 1) & 0x55555555: 存储奇数位，移动
		//到后面,清空偶数位。做或操作结合在一起。由ABCD EFGH -> BADC FEHG
		i = (i & 0x55555555) << 1 | (i >>> 1) & 0x55555555;
		//以4位为标准交换相邻2位的位置，首先&0x33333333(0011), 保留后两位，并
		//移动到前面.(i >>> 2) & 0x33333333,保留前两位并移动到后面。
		//由BADC FEHG ==> DCBA HGFE
		i = (i & 0x33333333) << 2 | (i >>> 2) & 0x33333333;
		//同理，以8位为标准移动位置，DCBA HGFE ==> HGFE DCBA
		i = (i & 0x0f0f0f0f) << 4 | (i >>> 4) & 0x0f0f0f0f;
		//i << 24，将最后8位bit启动到最前面(第一位)
		//((i & 0xff00) << 8),取倒数第三个8位bit并移动到第二位
		//((i >>> 8) & 0xff00)取第二个8位bit并移动到第三位, 
		//(i >>> 24)将第一个8位移动到第四位
		// A B C D (8位为基准) ==> D C B A 
		i = (i << 24) | ((i & 0xff00) << 8) |
			((i >>> 8) & 0xff00) | (i >>> 24);
		return i;
	}

	/**
	 * Returns the signum function of the specified {@code int} value.  (The
	 * return value is -1 if the specified value is negative; 0 if the
	 * specified value is zero; and 1 if the specified value is positive.)
	 *
	 * 传递的值如果为0返回0,正数返回1,负数返回-1
	 *
	 * @param i the value whose signum is to be computed
	 * @return the signum function of the specified {@code int} value.
	 * @since 1.5
	 */
	public static int signum(int i) {
		// HD, Section 2-7
		//0 ==> 0 | 0 = 0; 正数: 0 | 1 = 1 ,负数: -1 | 0 = -1
		//注意第一个是有符号右移(正数补0,负数补1),第二个为无符号右移(补0)
		return (i >> 31) | (-i >>> 31);
	}

	/**
	 * Returns the value obtained by reversing the order of the bytes in the
	 * two's complement representation of the specified {@code int} value.
	 *
	 * 按照byte进行转置
	 *
	 * @param i the value whose bytes are to be reversed
	 * @return the value obtained by reversing the bytes in the specified
	 *     {@code int} value.
	 * @since 1.5
	 */
	public static int reverseBytes(int i) {
		//直接调用reverse的最后一步，按8位byte进行置换
		return ((i >>> 24)           ) |
			   ((i >>   8) &   0xFF00) |
			   ((i <<   8) & 0xFF0000) |
			   ((i << 24));
	}

	/**
	 * Adds two integers together as per the + operator.
	 * 求和
	 * @param a the first operand
	 * @param b the second operand
	 * @return the sum of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static int sum(int a, int b) {
		return a + b;
	}

	/**
	 * Returns the greater of two {@code int} values
	 * as if by calling {@link Math#max(int, int) Math.max}.
	 * 比较较大值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the greater of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static int max(int a, int b) {
		return Math.max(a, b);
	}

	/**
	 * Returns the smaller of two {@code int} values
	 * as if by calling {@link Math#min(int, int) Math.min}.
	 * 比较较小值
	 * @param a the first operand
	 * @param b the second operand
	 * @return the smaller of {@code a} and {@code b}
	 * @see java.util.function.BinaryOperator
	 * @since 1.8
	 */
	public static int min(int a, int b) {
		return Math.min(a, b);
	}

	/** use serialVersionUID from JDK 1.0.2 for interoperability(互通性) */
	@Native private static final long serialVersionUID = 1360826667806852920L;
}
```
