# deconstruct
**deconstruct** provides a Pythonic analogue of C's `struct`, primarily for the purpose of interpreting (i.e. _deconstructing_) contiguous binary data.  
  
Internally, deconstruct uses Python's [struct module](https://docs.python.org/3/library/struct.html) and can be considered an abstraction of sorts. struct (the module) can be frustrating to use: its format strings appear arcane and furthermore separate the description of the data from its representation, a definite strength of C's `struct`.
  
In contrast, deconstruct allows structs to be defined and used using a syntax that is Pythonic while maintaining close correspondence to C.  
  
### Usage  
With deconstruct, the struct definition (adapted from [input.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input.h)):

```C  
struct input_event {  
    long time[2];
    short type;
    short code;
    int value;
};  
```  

can be represented as:

```Python
import deconstruct as c  
  
class InputEvent(c.Struct):  
    time: c.long[2]
    type: c.short
    code: c.short
    value: c.int
```
   
This definition can be used to interpret and then access binary data:

```Python  
>>> buffer = b'Some  arbitrary  buffer!'
>>> event = InputEvent(buffer)
>>> event.type
25120
>>> event.code
26229
>>> event.time[0]
8241904116577431379
>>> event.time[1]
2340027244253309282
>>> event.value
561145190 
```  

Of course, in reality the buffer passed in is more likely to come from something more useful, like a file. Notice that fixed-size arrays can be specified using the syntax `type[length]`, a further improvement on Python's struct.
  
### Installation  
deconstruct is currently available as a (tiny) wheel package. [Download](https://github.com/biqqles/deconstruct/releases/download/v0.1/deconstruct-0.1-py3-none-any.whl) and install with the following command:

```sh
python3 -m pip install deconstruct-0.1-py3-none-any.whl --user
```

Alternatively, simply clone this repository:

```sh
git clone https://github.com/biqqles/deconstruct.git
```

deconstruct has no dependencies but requires `Python >= 3.6` as it makes use of the class annotations added in that release (see [PEP 526](https://www.python.org/dev/peps/pep-0526/)).

## API listing

### Struct(buffer: bytes)
Subclass this to define your own structs. Subclasses should only contain field definitions of C types defined in this package (more on this [below](#c-types)).

When you instantiate your Struct with a `bytes` object, deconstruct creates a format string and uses it to unpack that buffer. In the instance, deconstruct types will be replaced with their equivalent Python types for use (e.g. `bytes` for `char`, `int` for `schar` and `float` for `double`).

#### Attributes
|Name            |Type       |Description  |
|----------------|-----------|-------------|
|`__byte_order__`|`ByteOrder`|Set this in your subclass definition to define the byte order used when unpacking the struct. One of:<ul><li>`ByteOrder.NATIVE` (default value)</li><li>`ByteOrder.BIG_ENDIAN`</li><li>`ByteOrder.LITTLE_ENDIAN`</li></ul>|
|`__type_width__`|`TypeWidth`|Set this in your subclass definition to define the type width and padding used for the struct. One of:<ul><li>`TypeWidth.NATIVE` (default value)</li><li>`TypeWidth.STANDARD`</li></ul>When `TypeWidth.NATIVE` is set, the struct will use native type widths and alignment. When `TypeWidth.STANDARD` is used, the struct will use Python's struct's "standard" widths<sup>[1](#f_st)</sup> and no padding.|

Note that `TypeWidth.NATIVE` can only be used with `ByteOrder.NATIVE`. This is a limitation of Python's struct.

#### Properties
|Name           |Type  |Description |  
|----------------|------|-------------|  
|`format_string` |`str` |The struct.py-compatible format string for this struct |
|`sizeof`        |`int` |The total size in bytes of the struct. Equivalent to C's `sizeof` |

### C types
deconstruct defines the following special types for use in Struct field definitions:<sup>[2](#f_ty)</sup>

|deconstruct type|C99 type            |Python format character|"Standard" width (bytes)<sup>[1](#f_st)</sup>|
|----------------|--------------------|-----------------------|------------------------|
|`char`          |`char*`             |`c`                    |1                       |
|`schar`         |`char`              |`b`                    |1                       |
|`uchar`         |`unsigned char`     |`B`                    |1                       |
|`short`         |`char`              |`h`                    |2                       |
|`ushort`        |`unsigned char`     |`H`                    |2                       |
|`int`           |`int`               |`i`                    |2                       |
|`uint`          |`unsigned int`      |`I`                    |2                       |
|`long`          |`long`              |`l`                    |4                       |
|`ulong`         |`unsigned long`     |`L`                    |4                       |
|`longlong`      |`long long`         |`q`                    |8                       |
|`ulonglong`     |`unsigned long long`|`Q`                    |8                       |
|`bool`          |`bool` (`_Bool`)    |`?`                    |1                       |
|`float`         |`float`             |`f`                    |4                       |
|`double`        |`double`            |`d`                    |8                       |
|`ptr`           |`void*`             |`P`                    |N/A\*                   |
|`size_t`        |`size_t`            |`n`                    |N/A\*                   |
|`ssize_t`       |`ssize_t`           |`N`                    |N/A\*                   |

<sup>\* only available with `__type_width__ = TypeWidth.NATIVE`.</sup>

As mentioned earlier, these types support a `type[length]` syntax to define fixed-size arrays. When a Struct is used to unpack a buffer, these types will resolve to a Python list of their equivalent types. You can also use these to define padding sequences.

---

<b id="f_st">1.</b> Python's struct has the concept of "standard" type sizes. This is somewhat confusing coming from C as its standards go to some length not to define a standard ABI. However, as this terminology is so fundamental to the documentation of Python's struct it is replicated here for simplicity's sake. **These sizes correspond with the *minimum* sizes [implied](https://en.wikipedia.org/wiki/C_data_types#Basic_types) for C's types.**

<b id="f_ty">2.</b> Because some of these conflict with Python's primitives, it is not recommended to `import * from deconstruct` as this will severely pollute your namespace (in fact this is a bad idea in general). I like to `import deconstruct as c` as shown above.
