# Base64 VLQ
这个概念是 source map 的核心，分为 Base64 编码和 VLQ 编码两种编码技术，本文主要围绕 VLQ 编码展开，如果您对 Base64 编码不熟悉，建议阅读 [Base64 维基百科](https://zh.wikipedia.org/wiki/Base64)。

## source map
### source map 的作用
在生产环境中，代码一般是以编译、压缩后的形态存在，对生产友好，但是调试时候定位的错误只能定位到编译压缩后的代码位置，此时的代码对人类的阅读很不友好，可能变量名被缩短失去语义，甚至是经过编译的，生产代码与开发代码已经没法一一对应。为了解决代码之间对应关系，人们设计了 source map 这种数据格式来。source map 就像一个索引表，将生产代码和开发代码联系起来，这样开发调试时就可以清晰的定位到开发代码中了。
#### 前端的编译处理过程包括但不限于
- 转译器/Transpilers（Babel）
- 编译器/Compilers（TypeScript，CoffeeScript，Webassembly）
- 压缩/Minifiers(UglifyJS)

这些都是可生成 source map 的操作。有了 source map，使得我们调试线上产品时，能直接看到开发环境的代码（需要浏览器提供支持）。
### source map 的格式
打开 source map，它大概是这个样子
``` JavaScript
    {
        version: 3,
        file: "outName.js",
        sourceRoot: "",
        sources: ["inputName1.js", "inputName2.js"],
        names: ["src", "maps", "are", "fun"],
        mappings: "XXXXX, XXXX, XXXXXX"
    }
```

属性对应的内容是

``` JavaScript
    - version: source map 的版本号。
    - file: 转换后的文件名。
    - sourcesRoot：转换前文件所在目录。如果输入和输出文件在同一个目录下，该项为空。
    - sources：转换前的文件。该项是一个数组，可能存在多个文件合并成一个文件。
    - names：转换前的所有变量名和属性名。
    - mappings：记录位置信息的字符串。
```

### mappings 的编码设计
每个位置使用五位，表示五个字段，从左边算起
- 第一位，表示这个位置在转换后的代码的第几列。
- 第二位，表示这个位置属于 sources 属性中的哪一个文件。
- 第三位，表示这个位置属于转换前代码的第几行。
- 第四位，表示这个位置属于转换前代码的第几列。
- 第五位，表示这个位置属于 names 属性中的哪一个变量。

#### 编码设计
最直观的想法是，将生成的文件中每个字符位置对应的原位置保存起来。

```
    “feel the force” ⇒ Yoda ⇒ “the force feel”
```

一个简单的文本转换输出，其中 Yoda 可以理解为一个转换器。将上面的输入与输出列成表格可以得出转换后输入与输出的对应关系。

|输出位置|输入|在输入中的位置|字符|
|-|-|-|-|
|行 1, 列 0|	Yoda_input.txt|	行 1, 列 5|	t|
|行 1, 列 1	|Yoda_input.txt	|行 1, 列 6|	h|
|行 1, 列 2	|Yoda_input.txt	|行 1, 列 7|	e|
|行 1, 列 4	|Yoda_input.txt	|行 1, 列 9|	f|
|行 1, 列 5	|Yoda_input.txt	|行 1, 列 10|o|
|行 1, 列 6	|Yoda_input.txt	|行 1, 列 11	|r|
|行 1, 列 7	|Yoda_input.txt	|行 1, 列 12	|c|
|行 1, 列 8	|Yoda_input.txt	|行 1, 列 13	|e|
|行 1, 列 10	|Yoda_input.txt	|行 1, 列 0|	f|
|行 1, 列 11	|Yoda_input.txt	|行 1, 列 1|	e|
|行 1, 列 12	|Yoda_input.txt	|行 1, 列 2|	e|
|行 1, 列 13	|Yoda_input.txt	|行 1, 列 3|	l|

将上面的表格整理记录成一个映射编码，看起来会是这样的：
```
    mappings(283 字符):1|0|Yoda_input.txt|1|5, 1|1|Yoda_input.txt|1|6, 1|2|Yoda_input.txt|1|7, 1|4|Yoda_input.txt|1|9, 1|5|Yoda_input.txt|1|10, 1|6|Yoda_input.txt|1|11, 1|7|Yoda_input.txt|1|12, 1|8|Yoda_input.txt|1|13, 1|10|Yoda_input.txt|1|0, 1|11|Yoda_input.txt|1|1, 1|12|Yoda_input.txt|1|2, 1|13|Yoda_input.txt|1|3
```

这样确实能实现映射效果，但这里源文件 `feel the force` 才 12 个有效字符，加上空格的长度也才 14，为了记录编译前后的映射文件就已经达到了 283 个字符。随着源文件的字符数增加，这个映射文件将会变得巨大。

##### 优化方案
1. 省去输出文件中的行号，改用 `;` 来标识换行

```
    mappings (245 字符): 0|Yoda_input.txt|1|5, 1|Yoda_input.txt|1|6, 2|Yoda_input.txt|1|7, 4|Yoda_input.txt|1|9, 5|Yoda_input.txt|1|10, 6|Yoda_input.txt|1|11, 7|Yoda_input.txt|1|12, 8|Yoda_input.txt|1|13, 10|Yoda_input.txt|1|0, 11|Yoda_input.txt|1|1, 12|Yoda_input.txt|1|2, 13|Yoda_input.txt|1|3;
```
2. 可符号化字符的提取

|序号|符号|
|-|-|
|0|the|
|1|force|
|2|feel|

搭配一个包含所有符号的数组：
```
name: ['the', 'force', 'feel']
```

在记录时，只需要记录一个索引，还原时通过读取 `names` 数组即可还原。
所以 `the` 的映射有原来的
```
0|Yoda_input.txt|1|5, 1|Yoda_input.txt|1|6, 2|Yoda_input.txt|1|7
```
简化为：
```
0|Yoda_input.txt|1|5|0
```
考虑到合并多文件打包的情况，输入文件也许不止一个，加入一个输入文件的记录索引
```
sources: ['Yoda_input.txt']
```
进一步将 `the` 的标示简化为：
```
0|0|1|5|0
```

到这一步，我们的 source map 文件大致的内容为：
```
    {
        version: 3,
        file: "Yoda_ouput.txt",
        sourceRoot: "",
        sources: ["Yoda_input.txt"],
        names: ["the", "force", "feel"],
        mappings(31 字符): 0|0|1|5|0, 4|0|1|9|1, 10|0|1|0|2;
    }
```

3. 记录相对位置
当文件内容巨大时，上面精简后的代码也有可能某些数字会随着增加而变得很长，如果一行的位置记录了某个位置，那么根据这一位置进行相对定位是可以到达一行内的任意位置。

具体到本例中，看看最初的表格中，记录的输出文件的位置：

|输出位置|输出位置|
|-|-|
|行 1, 列 0|	行 1, 列 0|
|行 1, 列 4|	行 1, 列 (上一值 + 4 = 4)|
|行 1, 列 10|	行 1, 列 (上一值 + 6 = 10)|

对应到整个表格则是：

|输出位置|	输入文件的索引|	输入的位置|	符号索引|
|-|-|-|-|
|行 1, 列 0|	0	|行 1, 列 5|	0|
|行 1, 列 +4|	+0	|行 1, 列 +4|	+1|
|行 1, 列 +6|	+0	|行 1, 列 -9|	+1|

到这一步，我们的 source map 文件大致的内容为：
```
    {
        version: 3,
        file: "Yoda_ouput.txt",
        sourceRoot: "",
        sources: ["Yoda_input.txt"],
        names: ["the", "force", "feel"],
        mappings(31 字符): 0|0|1|5|0, 4|0|1|4|1, 6|0|1|-9|1;
    }
```

观察编译后文件的内容，可以看出，通常是被压缩至几行代码内，这样通过 `;` 就可以标识行号。上面介绍 source map 格式的时候，就已经提及了第二种优化，source map 文件中使用 names 记录源文件中的变量名和属性名，只需要读取这个数组的下标就可以映射出源文件和目标文件的变量。源文件中的代码都是连续的，只要记录了起始位置，通过相对位置的记号即可到达文件中的任意位置。通过前三种优化已经能有效减少 mappings 的字符数量，如果文件体积不是特别巨大，到这一步已经能够满足应用了。对于内存控制精确到字节的工程师，只有 mappings 中的分隔符，以及如何记录大数字这两个优化方向了。

#### VLQ 编码
1. VLQ 以数字的方式呈现

如果我们提前知晓要记录的数字每一位的个数，就可以省略间隔符。比如：
```
假如要记录的数字是
1|2|3|4
那么直接记录成如下也能被正确识别
1234
```
如果我们使用下划线来标识一个数字后是否跟有其他数字：
1<u>2</u>3<u>45</u>67
解读规则为：
- 1没有下划线，那解析出来的第一个数字是1
- 2有下划线，继续解析，遇到3,3没有下划线，第二个数字解析结束，是23
- 4有下划线，继续解析，5有下划线，继续解析，6没有下划线，第三个数字解析结束，是456
- 7没有下划线，第四个数字是7


2. VLQ 以二进制的方式呈现

在二进制系统中，我们使用 6 个字节来记录一个数字，用其中一个字节来标识它是否结束（下方 C），再用一位标识正负（下方 S），剩下还有四位用来表示数值。用这样6个字节来表示我们需要的数字。

<table>
    <tr>
        <td>B5</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0</td> 
   </tr>
    <tr>
        <td>C</td>
        <td colspan="4">Value</td>    
        <td>S</td>    
    </tr>
</table>

任意数字中，第一组的第一个字节就已经明确标明该数字的正负，所以后续字节不需要再标识，也就是说第一组有 4 个字节来表示数值，后续每一组都有 5 个字节来表示数值（每组的第一个字节标识是否结束）

现在我们用二进制规则来编码之前的这个数字序列 `1|23|456|7`。

将数字转换对应的真值二进制
<table>
    <tr>
        <td>数值</td> 
        <td>二进制</td> 
   </tr>
    <tr>
        <td>1</td> 
        <td>1</td> 
   </tr>
    <tr>
        <td>23</td>   
        <td>10111</td>   
    </tr>
    <tr>
        <td>456</td>   
        <td>1111001000</td>   
    </tr>
    <tr>
        <td>7</td>   
        <td>111</td>   
    </tr>
</table>

- 对 1 进行编码
1 需要一位来表示，首个字节组有四位来表示值
<table>
    <tr>
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td> 
   </tr>
    <tr>
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>1</td>   
        <td>0</td>   
    </tr>
</table>

- 对 23 进行编码
23 的二进制位 10111 一共需要5位，只能拆分成两组，第一组截取后四位，剩下一位放入第二组中。
<table>
    <tr>
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td>
        <td></td> 
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td> 
   </tr>
    <tr>
        <td>1</td>   
        <td>0</td>   
        <td>1</td>   
        <td>1</td>   
        <td>1</td>   
        <td>0</td>   
        <td></td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>1</td>   
    </tr>
</table>

- 对 456 进行编码
456 的二进制 111001000 需要 9 个字节，同样第一组截取后四位，剩下的每五位截取放入到跟随的字节组中即可。
<table>
    <tr>
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td>
        <td></td> 
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td> 
   </tr>
    <tr>
        <td>1</td>   
        <td>1</td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td>0</td>   
        <td></td>   
        <td>0</td>   
        <td>1</td>   
        <td>1</td>   
        <td>1</td>   
        <td>0</td>   
        <td>0</td>   
    </tr>
</table>

- 对 7 进行编码
7 的二进制为 111，只需要一个字节组
<table>
    <tr>
        <td>B5(C)</td> 
        <td>B4</td> 
        <td>B3</td> 
        <td>B2</td> 
        <td>B1</td> 
        <td>B0(S)</td> 
   </tr>
    <tr>
        <td>0</td>   
        <td>0</td>   
        <td>1</td>   
        <td>1</td>   
        <td>1</td>   
        <td>0</td>   
    </tr>
</table>

将上面编码合并得到最终的编码：
```
    000010 101110 000001 110000 011100 001110
```

之所以用 6 位来作为一组字节组的长度，是因为 6 位的长度可以进行 base64 编码，将上面得到的最终编码对照 base64 编码表，转换的结果就是
```
    CuBwcO
```

#### 利用 Base64 VLQ 编码生成最终的 source map
还记得我们上面讨论中的示例，优化过的结果是
```
    {
        version: 3,
        file: "Yoda_ouput.txt",
        sourceRoot: "",
        sources: ["Yoda_input.txt"],
        names: ["the", "force", "feel"],
        mappings(31 字符): 0|0|1|5|0, 4|0|1|4|1, 6|0|1|-9|1;
    }
```
现在进行 Base VlQ 编码，先转成二进制，再转 VLQ 表示。
```
0 -> 0 -> 000000
0 -> 0 -> 000000
1 -> 1 -> 000010
5 -> 101 -> 001010
0 -> 0 -> 000000
```
合并后的编码为：
```
000000 000000 000010 001010 000000
```
转 Base64 后得到：
```
AACKA
```
经过 Base64 VLQ 编码后的 source map 结果是：
```
    {
        version: 3,
        file: "Yoda_ouput.txt",
        sourceRoot: "",
        sources: ["Yoda_input.txt"],
        names: ["the", "force", "feel"],
        mappings(31 字符): AACKA, IACIC, MACTC
    }
```
上面的结果就是 source map 的核心原理，通过数据设计和编码转换，达到用最简便的方式记录信息，而且考虑到生成的文件体积大小问题，可以说是设计思想是相当巧妙。

## 代码实现
以下是 Base64 VQL 源码实现，是GitHub 上[MattiasBuelens](https://github.com/MattiasBuelens)实现的，用的 TypeScript，这里我将他实现转换成 JavaScript，并且标注上我自己的理解。其中位运算可以参考这篇[文章](https://github.com/D-kylin/note/blob/master/0%E5%92%8C1%E7%BC%96%E7%A0%81%E7%9A%84%E8%83%8C%E5%90%8E.md)，如果 Base64 编码不熟悉的建议先查看[Base64维基百科](https://zh.wikipedia.org/wiki/Base64)。
```JavaScript
let charToInteger = {};
let integerToChar = {};

// Base64 和数字互转
'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/='.split( '' ).forEach( function ( char, i ) {
	charToInteger[ char ] = i;
	integerToChar[ i ] = char;
});

function decode(string) { //Base64 转数字
	let result = [];
	let shift = 0;
	let value = 0;

	for (let i = 0; i < string.length; i++) {
		let integer = charToInteger[string[i]];

		if (integer === undefined) {
			throw new Error(`Invalid character ${string[i]}`)
		}

		const hasContinuationBit = integer & 32; //小于 32 的数跟 32 通过与位运算都是 0

		integer &= 31; // 31 的二进制编码是 11111，相当于取这个数值二进制编码的右五位。
		value += integer << shift; //无符号左移运算，每一位向左移动 n 位，然后在移动前位置补 n 个 0,这个 shift 的值是根据上一个字符决定是否左移，即如果上一个符号的 hasContinuationBit 大于 0，表示这个数值还没有读取完毕，需要继续读取下一个符号的值加起来才是完整值。<< 的优先级比 += 要高，先 << 后 +=。

		if (hasContinuationBit) {
			shift += 5; //左移的位数加 5，因为如果需要分割的时候，后面的字节组都是有 1 个字节表示是否结束，5 个字节表示值，所以左移值是五位。
		} else {
			const shouldNegate = value & 1; //如果读取完毕，判断这个值的二进制的末位是否 1，如果是 1 表明数值是负数，如果是 0 表明数值是正数 
			value >>= 1; //因为最后一位是正负数的记号标识，所以这个值需要右移一位。

			result.push(shouldNegate ? -value : value);

			//reset
			value = shift = 0;
		}
	}

	return result;
}

function encode(value) { //数字转 Base64
	let result;

	if (typeof value === 'number') {
		result = encodeInteger(value);
	} else {
		result = '';
		for(let i = 0; i < value.length; i++) {
			result += encodeInteger(value[i])
		}
	}

	return result;
}

function encodeInteger(num) { //数字转 Base64 核心代码
	let result = '';

	if (num < 0) {
		num = (-num << 1) | 1; //如果要转的数字小于 0，则左移一位，并且将末位变为 1，来标识负数
	} else {
		num <<= 1; //如果要转的数字大于 0，则左移一位，末位自动补 0
	}

	do {
		let clamped = num & 31; //取二进制编码的后五位
		num >>= 5; //然后将数值左移五位

		if (num > 0) { //判断左移后的值是否大于 0，如果还有大于 0 的值，表面该数值还没转完，如果等于 0，表面该数值已完全转为 Base64 字母
			clamped |= 32; //如果进入这个条件语句，表示该数值后还会有值，这组字节组的首个字节一定是 1，所以需要读取的值是六位。通过 |= 跟 32 位运算可以将值变为 6 位二进制值。
		}

		result += integerToChar[clamped];
	} while (num > 0); //左移五位后判断 num 是否大于 0，如果是表明值还没读取完毕，如果不是，结束循环。

	return result;
}
```

## 结语
以上代码便是该作者的实现思路，主要是通过位运算。里面用的数值 5，31，32 这些值看数值反而不是很直观，如果将他们转换成“真值”二进制码，更容易看出实现思路。这个代码实现结合 VQL 编码的设计思想我认为能更好的理解这种编码格式。现在各种算法都有框架或者底层来实现封装，大多数时候都是直接拿来就用，理解其实现原理大多数情况下也是帮助不大。不过我认为在学习一个技术的时候，了解其当初的设计思想和实现原理对编程思想的提高很有帮助。这篇文章如果有错误的地方欢迎指教，感谢您的阅读。

#### 参考资料，感谢以下文章的作者。
> [source map 的原理探究](https://github.com/wayou/wayou.github.io/issues/9)

> [Source Maps under the hood – VLQ, Base64 and Yoda](https://blogs.msdn.microsoft.com/davidni/2016/03/14/source-maps-under-the-hood-vlq-base64-and-yoda/#comment-626)

> [JavaScript Source Map 详解](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)