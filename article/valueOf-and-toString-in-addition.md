# 加法运算中的 `valueOf` 和 `toString` 
假设有代码如下：
```javascript
let obj = {
    toString() {
        console.log('toString');
        return 'obj';
    },
    valueOf() {
        console.log('valueOf');
        return {}; // 返回空对象
    }
};

alert('a' + obj); // 打印valueOf toString，执行结果为aobj
alert(1 + obj); // 打印valueOf toString，执行结果为1obj

let obj1 = {
    toString() {
        console.log('toString');
        return 'obj1';
    },
    valueOf() {
        console.log('valueOf');
        return 12; // 返回数字
    }
};

alert('a' + obj1); // 打印valueOf，执行结果为a12
alert(1 + obj1); // 打印valueOf，执行结果为13
```


上述代码的执行结果为什么会各不相同，到底什么时候该调用 `valueOf` ？ `toString` 呢？二者的 **优先级** 呢？为什么还有 **两个都调用** 的情况？对象的 `valueOf` 以及 `toString` 方法的触发机制比较复杂，结合网上[相关回答文章](https://stackoverflow.com/questions/2485632/valueof-vs-tostring-in-javascript)以及[ECMA规范](http://www.ecma-international.org/ecma-262/5.1/)后，大概尝试解答一下疑惑。在这之前，我们需要先了解几个概念


## primitive values
在JS中，除 `Object` 以外的 **原始(基本)类型** 的值本身是无法被改变的。以字符串为例，对字符串的操作返回的一定是新的字符串，而不是对原字符串进行修改，原字符串本身是没有被改变的。像这样的类型对应的值，我们称之为 **primitive values** ，也就是原始值。

## `ToPrimitive` 
[抽象方法](http://www.ecma-international.org/ecma-262/5.1/#sec-9.1)，接受 `input` 参数和可选的 `PreferredType` 参数，会将 `input` 参数转换为**非对象类型**(none-Object type)。根据可选参数（文档里叫 _hint_ ，中文翻译是**暗示**，我这里为了方便理解就写成了参数） `PreferredType` 来决定将 `input` 转换成哪种类型：

- 参数类型为 `Undefined` 、 `Null` 、 `Boolean` 、 `Number` 、 `String` 的转换结果就等于 **input** 参数
- 参数类型为 `Object` 的则比较特殊，它会返回一个**默认值(default value)**，通过对象的内置方法 `[[DefaultValue]]` 以及该方法对应的可选参数 `PreferredType` 来检索这个默认值



## `ToNumber` 
[抽象方法](http://www.ecma-international.org/ecma-262/5.1/#sec-9.3)，接受 `input` 参数，会将 `input` 参数转换为 `Number` 类型的值，转换规则如下表：

| 参数类型 | 转换结果 |
| --- | --- |
| Undefined | **NaN** |
| Null | **+0** |
| Boolean | **false** 转换为 **+0**，**true** 转换为 **1** |
| Number | 与 `input` 参数一致 |
| String | 比较复杂，感兴趣的可以[参考文档](http://www.ecma-international.org/ecma-262/5.1/#sec-9.3.1) |
| Object | 转换步骤如下：<br>1. Let _primValue_ be ToPrimitive(input argument, hint Number). 先采用 _hint_ 为 `Number` 的 `ToPrimitive` 转换输入值<br>2. Return ToNumber(_primValue_). 然后将转换后的值再进行 `ToNumber` 操作 |



## `ToString` 
[抽象方法](http://www.ecma-international.org/ecma-262/5.1/#sec-9.3)，接受 `input` 参数，会将 `input` 参数转换为 `String` 类型的值，转换规则如下表：

| 参数类型 | 转换结果 |
| --- | --- |
| Undefined | **"undefined"** |
| Null | **"null"** |
| Boolean | **true** 转换为 **"true"**，**false** 转换为 **"false"** |
| Number | 比较复杂，感兴趣的可以[参考文档](http://www.ecma-international.org/ecma-262/5.1/#sec-9.8.1) |
| String | 与 `input` 参数一致 |
| Object | 转换步骤如下：<br>1. Let _primValue_ be ToPrimitive(input argument, hint String).先采用 _hint_ 为 `String` 的 `ToPrimitive` 转换输入值<br>2. Return ToString(_primValue_). 然后将转换后的值再进行 `ToString` 操作 |



## `[[DefaultValue]](hint)` 
[对象内置方法](http://www.ecma-international.org/ecma-262/5.1/#sec-8.12.8)，根据提供的 _hint_ 类型执行不同的处理步骤，规则比较复杂，这里只提下 _hint_ **不存在** 以及 _hint_ 为 `Number` 类型时的处理规则：

- 当对象的内置方法 `[[DefaultValue]]` 没有明确提供 _hint_ 时，一般都把 _hint_ 当成 `Number` 进行处理。 `Date` 类型对象**除外**， `Date` 类型一般把它的 _hint_ 当成是 `String` 来处理
- 如果提供的 _hint_ 为 `Number` ，那么：
    1. 如果对象自身存在 `valueOf` 方法，则调用自身的 `valueOf` 方法：
        1. 返回值为 **primitive value** ，则返回该结果
        2. 否则忽略该值，继续往下判断
    2. 如果对象自身存在 `toString` 方法，则调用自身的 `toString` 方法：
        1. 返回值为 **primitive value** ，则返回该结果
        2. 否则忽略该值，继续往下判断
    3. 抛出异常 `TypeError` 
- 以上规则只针对 ***native objects***(原生对象，未经修改的对象)生效。如果一个***host object***自己实现(implements)了它的内置方法 `[[DefaultValue]]` ，那它必须确保方法的返回值是一个***primitive value***(基本类型值)



## `+` 号运算符
`+` 号运算符可以执行字符串拼接或者数字相加操作。以 `a + b` 为例：

1. 获取 `ToPrimitive(a)` 的返回值 `primA` 
2. 获取 `ToPrimitive(b)` 的返回值 `primB`
3. 如果 `primA` **或者 **`primB` 类型为 `String` ，则：
    1. 返回 `ToString(primA)` 和 `ToString(primB)` 串联而成的字符串，注意：这里调用的是抽象方法 `ToString` ，而不是 `toString` 
4. 否则返回对 `ToNumber(primA)` 和 `ToNumber(primB)` 应用加法运算的结果



## 执行过程分析
### `alert('a' + obj)` 

1. `toPrimitive('a')` 返回值为 `'a'` 
2. 因为 `obj` 是个对象，所以执行 `ToPrimitive(obj)` 时会调用 `[[DefaultValue]]` 进行取值， `hint` 缺省则默认为 `Number` ，执行 `valueOf` 打印 `'valueOf'` ，返回值为对象(**非primitive value**)，继续执行 `toString` 打印 `'toString'` ，返回 `'obj'` 
3. `'a'` 为 `String` 类型，则执行 `ToString('a') + ToString('obj')` ，返回值为 `'aobj'` 



### `alert(1 + obj)` 

1. `toPrimitive(1)` 返回值为 `1`
2. 执行 `ToPrimitive(obj)` 过程跟上面一致，打印 `'valueOf' 'toString'` 返回 `'obj'` ，不再赘述
3. `'obj'` 为 `String` 类型，则执行 `ToString(1) + ToString('obj')` ，返回值为 `'1obj'`



### `alert('a' + obj1)`

1. `toPrimitive('a')` 返回值为 `'a'`
2. 因为 `obj1` 是个对象，所以执行 `ToPrimitive(obj1)` 时会调用 `[[DefaultValue]]` 进行取值， `hint` 缺省则默认为 `Number` ，执行 `valueOf` 打印 `'valueOf'` ，返回值为 `12` 为 **primitive value** ，直接返回
3. `'a'` 为 `String` 类型，则执行 `ToString('a') + ToString(12)` ，返回值为 `'a12'`



### `alert(1 + obj1)`

1. `toPrimitive(1)` 返回值为 `1`
2. 执行 `ToPrimitive(obj1)` 过程跟上面一致，打印 `'valueOf'` 返回 `12` ，不再赘述
3. `1` 和 `12` 都不为 `String` 类型，则执行 `ToNumber(1) + ToNumber(12)` ，返回值为 `13`



## 总结
`+` 号运算符处理的是**两侧变量 `ToPrimitive` 后的结果**，所以核心点在于 `Object` 类型对象的转换。而加法运算获取 `[[DefaultValue]]` 时， `Object` 类型对象的缺省 _hint_ 值为 `Number` ，所以会优先调用 `valueOf` ，在其返回值不是 **primitive value** 的情况下才会继续调用 `toString` 。而 `+` 号两侧的 `ToPrimitive` 结果**必须都为** `Number` 类型才会执行数字相加操作，否则就进行字符串的拼接(串联)
