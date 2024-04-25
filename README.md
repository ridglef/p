Obfuscation is the practice of obscuring code to be more confusing and unclear to a reverse engineer.  This allows developers to defend against unauthorized access to sensitive code.  The Java Compiler (javac) translates source code into a lower level set of instructions that can be run universally via the Java Virtual Machine (jvm).  We can use a disassembler to change the instructions in a classfile (compiled java file) for the purposes of obfuscation.  This write-up will provide a basic overview of some common obfuscation techniques for obscuring the values of shorts (2-byte numbers).

#### Pushing A Short to the stack

To begin, we must first demonstrate how shorts are loaded to the jvm stack. For example, we will use the short `9999`

```
sipush 9999
```


#### Primitive Bitwise Instructions

The most common and robust way to obfuscate shorts is to use the primitive bitwise instructions. 

```
	sipush -27282 // Load Short
	iconst_m1 // Load -1 for the Bitwise NOT
	ixor // Xor the short by -1 // Disregard this number for a minute
	sipush 19604  // Load Short
	sipush 16782 // Load Short
	ior // Bitwise OR the 2 above shorts
	ixor // Xor the disgregarded number by the nunber created from the IOR opeation
```

We can represent this in java source code with the following expression `~-27282 ^ (0x4C94 | 0x418E);` which results in our original short of 9999.

#### Bit Shifting

Another (less) common way to obfuscate shorts is to use the bitwise shift operators.  These are often disregarded because they can be hard to work with without data loss.  We can however, use multiples of 16 (16,32,48,64.....) without any loss.  In this example we will shift the short left by 48 and then use that same shift with a right shift zero fill operation.

```
	ldc 655294464 // Load INTEGER result of the shift
	bipush 48 // Load BYTE shift
	iushr // Right shift zero fill
```

We can represent this in java source code with the following expression `655294464 >>> 48;` which again results in our original short of 9999.

#### String Length Computation

The second most common strategy for obfuscation shorts is to load a string  to the stack and add or subtract its length.  In this example we will load a 10 character string and add it to (9999 - 10).

```
	sipush 9989 // Load the short minus 10
	ldc  "          " // Load a 10 character String
	invokevirtual java/lang/String.length ()I  // Invoke string.Length()
	iadd // Add the string length to 9989
```

We can represent this in java source code with the following expression `9989 + "          ".Length();` which again results in our original short of 9999.  This is starting to get repetitive, and it is now assumed that operations will return 9989.

#### Min/Max

Another common strategy is to compare 2 shorts with the Math class min/max methods.

```
	sipush 9968 // Load Short (smaller)
	sipush 9999 // Load Short (bigger)
	invokestatic java/lang/Math.max (II)I // Invoke max method, the bigger short wins
	sipush 10027 // Load Short (bigger)
	sipush 9973 // Load Sort (smaller)
	invokestatic java/lang/Math.max (II)I // Invoke max method, the bigger short wins
	invokestatic java/lang/Math.min (II)I // Invoke min method, the smaller short wins
```

The order can be mix-matched in any combination as long as the route follows back to the original short.  This can be represented in java source code with `Math.min(Math.max(9968, 9999), Math.max(10027, 9973));`

#### Breaking Down

At their core, shorts are a u2 value (2-bytes), which is simply 2 u1 (1-byte) values. A very rare strategy yet elegantly simple is to separate the u2 into 2 u1s and use some bitwise magic to reconstruct them. 

For this strategy we will represent this in java source code for simplicity.  We will use an array of bytes to improve the scalability.

```java
public class Main {
    public static final byte[] u1A = new byte[2];
    
    public static void main(String[] args) {
        short ourShortValue = (short)((u1A[0] & 0xFF) << 8 | u1A[1] & 0xFF); // Reconstruct the short
    }

	 static {
        Main.u1A[0] = 39; // Put the first byte
        Main.u1A[1] = 15; // Put the second byte
    }
}
```

#### Unsafety

A relatively-uncommon strategy is to load the short through `sun.misc.Unsafe`.  This alone does not provide any protection and one can view it more as a control-flow operation.  We will again represent this through source code.

```java
public static final long[] mems = new long[1];  
public static Field unsafeField;  
public static Unsafe unsafe;  
  
static {  
    try {  
        unsafeField = Unsafe.class.getDeclaredField("theUnsafe");  
        unsafeField.setAccessible(true);  
        unsafe = (Unsafe) unsafeField.get(null);  
    } catch (Exception ignored) {}  
        mems[0] = unsafe.allocateMemory(2L);  // Allocate 2 bytes of memory
        unsafe.putShort(mems[0], (short) 4312);  // Push our short
}  
  
public static void main(String[] args) {  
	short ourShortValue = unsafe.getShort(mems[0]));  // Load our short
}
```


#### Reverse Bytes

A moderately-common strategy is to simply reverse the bytes of a Short with the reverseBytes method. 

```
	sipush 3879 // load reversed bytes of 9999
	invokestatic java/lang/Short.reverseBytes (S)S // reverse bytes
```

We can represent this in java source code with `Short.reverseBytes(3879);`.

### In Practice

While each strategy is quite primitive by itself, all of these method lay on top of each other to provide a fair wall of defense.

A decompilation of these strategies in practice:
[![[[decomp.png]](https://raw.githubusercontent.com/ridglef/p/main/decomp.png)]](https://raw.githubusercontent.com/ridglef/p/main/decomp.png)

All of these defenses still remain weak if you only focus on them itself.  Other transformations (namely flow obfuscation and nativefying the constant pool) will complement Short obfuscation greatly and significantly increase the difficulty of retrieving the shorts.
