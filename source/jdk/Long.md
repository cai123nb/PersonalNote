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
				//取前len - 1位, 如果可以正常转换(没有抛出跨界异常), 说明前i-1位是在有符号范围内的,即2进制63位
                long first = parseLong(s.substring(0, len - 1), radix);
				//取最后一位
                int second = Character.digit(s.charAt(len - 1), radix);
                if (second < 0) { //最后一位不能越界
                    throw new NumberFormatException("Bad digit at end of " + s);
                }
				//此处存疑, 如正常LONG的最大值: 7FFF_FFFF_FFFF_FFFF, 
				//对应最大的Unsigned LONG为: FFFF_FFFF_FFFF_FFFF, 但是如果传递7_FFFF_FFFF_FFFF_FFFF,仍然可以正常解析
				//因为compareUnsigned为0, first,secend正常,溢出后的result和first相同. 对于4_0000_0000_0000_0DEF, 则会报错, 
				//因为溢出后的result为正数, 比first小, 不能满足要求. 这里需要进一步的确定.
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
 }
```