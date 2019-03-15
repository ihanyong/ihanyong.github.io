2019-03-06-【Java基础知识梳理】Java位运算的应用（hex转码）


先看代码，实现了两个方法。hexEncode() 把byte数组转换为16进制表示的char数组；hexDecode()相反，把16进制表示的char数组转回到byte数组的形式。


# 基本类型的大小
1 byte = 8 bit
byte : 8 bit
short: 16 bit
int: 16 bit
long: 16 bit
float: 16 bit
double: 16 bit
boolean: JVM 规范定义的为 boolean 点用4个字节， boolean 数组中每个元素为1个字节。 对于虚拟机来说不存在boolean类型， 实现中用 int 来代替。 为什么不用更短的 byte,short 等？  因为在硬件层面，对当下32位的处理器来说，CPU一次处理的数据是32位。


# 补码的形式


```java

package olddog.digest;

/**
 * HexUtils
 *
 * @author yong.han
 * 2019/3/14
 */
public class HexUtils {

    private static final char[] DIGITS = "0123456789ABCDEF".toCharArray();

    public static char[] hexEncode(byte[] bts) {
        char[] cs = new char[bts.length << 1];

        int j = 0;
        for (int i = 0; i < bts.length; i++) {
            cs[j++] = DIGITS[(0xF0 & bts[i]) >>> 4];
            cs[j++] = DIGITS[(0x0F & bts[i])];
        }
        return cs;
    }

    public static byte[] hexDecode(char[] cs) {
        if (cs.length % 2 == 1) {
            throw new IllegalArgumentException("char 数组长度应该为偶数");
        }

        byte[] result = new byte[cs.length >> 1];

        int j = 0;
        for (int i = 0; i < result.length; i++) {
            int t = Character.digit(cs[j++], 16) << 4;
            t = t | (Character.digit(cs[j++], 16));
            result[i] = (byte) (t & 0xFF);
        }

        return result;
    }

}

```


Junit Test Case
```java
package olddog.digest;

import org.junit.Assert;
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.ExpectedException;

import static org.junit.Assert.assertArrayEquals;

/**
 * HexUtilsTest
 *
 * @author yong.han
 * 2019/3/14
 */
public class HexUtilsTest {
    public static class Tuple2 {
        char[] f0;
        byte[] f1;

        public static Tuple2 of(char[] f0, byte[] f1) {
            Tuple2 tuple2 = new Tuple2();
            tuple2.f0 = f0;
            tuple2.f1 = f1;
            return tuple2;
        }
    }

    private static Tuple2[] simples = new Tuple2[]{
            Tuple2.of(new char[]{'0', '1'}, new byte[]{1})
            ,Tuple2.of(new char[]{'0', '1', '2', '3', '4', '5'}, new byte[]{1, 35, 69})
            , Tuple2.of(new char[]{'F', 'D', 'E', 'C', 'B', 'A'}, new byte[]{-3, -20, -70})
            , Tuple2.of(new char[]{'0', 'F', 'E', '8', '9', 'C'}, new byte[]{15, -24, -100})
            , Tuple2.of(new char[]{'9', 'C'}, new byte[]{-100})
            , Tuple2.of(new char[]{'0', '1', '2', '3', '4', '5','F', 'D', 'E', 'C', 'B', 'A','0', 'F', 'E', '8', '9', 'C'}, new byte[]{1, 35, 69,-3, -20, -70,15, -24, -100})
    };

    @Test
    public void testHexEncode() {
        for (Tuple2 simple : simples) {
            assertArrayEquals(simple.f0, HexUtils.hexEncode(simple.f1));
        }
    }

    @Test
    public void testHexDecode() {
        for (Tuple2 simple : simples) {
            assertArrayEquals(simple.f1, HexUtils.hexDecode(simple.f0));
        }
    }


    @Rule
    public ExpectedException expectedException = ExpectedException.none();
    @Test
    public void testHexDecode_exception() {
        expectedException.expect(IllegalArgumentException.class);
        expectedException.expectMessage("char 数组长度应该为偶数");
        HexUtils.hexDecode(new char[]{'a','a','a'});

    }
}

```