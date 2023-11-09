## Java 中@SuppressWarnings("unchecked") 的作用



**简介：**java.lang.SuppressWarnings是J2SE5.0中标准的Annotation之一。可以标注在类、字段、方法、参数、构造方法，以及局部变量上。
**作用：**告诉编译器忽略指定的警告，不用在编译完成后出现警告信息。
**使用：**
@SuppressWarnings(“”)
@SuppressWarnings({})
@SuppressWarnings(value={})
**根据 sun的官方文档描述：**
value -将由编译器在注释的元素中取消显示的警告集。允许使用重复的名称。忽略第二个和后面出现的名称。出现未被识别的警告名*不是*错误：编译器必须忽略无法识别的所有警告名。但如果某个注释包含未被识别的警告名，那么编译器可以随意发出一个警告。

各编译器供应商应该将它们所支持的警告名连同注释类型一起记录。鼓励各供应商之间相互合作，确保在多个编译器中使用相同的名称。

**示例：**

 @SuppressWarnings("unchecked")

告诉编译器忽略 unchecked 警告信息，如使用List，ArrayList等未进行参数化产生的警告信息。

 @SuppressWarnings("serial")

如果编译器出现这样的警告信息：The serializable class WmailCalendar does notdeclare a static final serialVersionUID field of type long 使用这个注释将警告信息去掉。

@SuppressWarnings("deprecation")

如果使用了使用@Deprecated注释的方法，编译器将出现警告信息。 使用这个注释将警告信息去掉。

 @SuppressWarnings("unchecked", "deprecation")

告诉编译器同时忽略unchecked和deprecation的警告信息。

@SuppressWarnings(value={"unchecked", "deprecation"})

等同于@SuppressWarnings("unchecked", "deprecation")　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　

 编码时我们总会发现如下变量未被使用的警告提示：

![image-20210309111523317](https://gitee.com/wanxianbo/pic-bed/raw/master/img/2021/03/20210309111524.png)

 上述代码编译通过且可以运行，但每行前面的“感叹号”就严重阻碍了我们判断该行是否设置的断点了。这时我们可以在方法前添加 @SuppressWarnings("unused") 去除这些“感叹号”。

**二、@SuppressWarings注解**　　　　　　　　　　　　　　　　　　　　　　　　　

**作用：用于抑制编译器产生警告信息。**

 示例1——抑制单类型的警告：

@SuppressWarnings("unchecked")

public void addItems(String item){

 @SuppressWarnings("rawtypes")

  List items = new ArrayList();

  items.add(item);

}

 示例2——抑制多类型的警告：

@SuppressWarnings(value={"unchecked", "rawtypes"})

public void addItems(String item){

  List items = new ArrayList();

  items.add(item);

}

 示例3——抑制所有类型的警告：

@SuppressWarnings("all")

public void addItems(String item){

  List items = new ArrayList();

  items.add(item);

}

**三、注解目标**　　　　　　　　　　　　　　　　　　　　　　　　　　　　　

 通过 @SuppressWarnings 的源码可知，其注解目标为类、字段、函数、函数入参、构造函数和函数的局部变量。

 而家建议注解应声明在最接近警告发生的位置。

**四、抑制警告的关键字**　　　　　　　　　　　　　　

| 关键字                   | 用途                                                         |
| ------------------------ | ------------------------------------------------------------ |
| all                      | to suppress all warnings（抑制所有警告）                     |
| boxing                   | to suppress warnings relative to boxing/unboxing operations（要抑制与箱/非装箱操作相关的警告） |
| cast                     | to suppress warnings relative to cast operations（为了抑制与强制转换操作相关的警告） |
| dep-ann                  | to suppress warnings relative to deprecated annotation（要抑制相对于弃用注释的警告） |
| deprecation              | to suppress warnings relative to deprecation（要抑制相对于弃用的警告） |
| fallthrough              | to suppress warnings relative to missing breaks in switch statements（在switch语句中，抑制与缺失中断相关的警告） |
| finally                  | to suppress warnings relative to finally block that don’t return（为了抑制警告，相对于最终阻止不返回的警告） |
| hiding                   | to suppress warnings relative to locals that hide variable（为了抑制本地隐藏变量的警告） |
| incomplete-switch        | to suppress warnings relative to missing entries in a switch statement (enum case)（为了在switch语句（enum案例）中抑制相对于缺失条目的警告） |
| nls                      | to suppress warnings relative to non-nls string literals（要抑制相对于非nls字符串字面量的警告） |
| null                     | to suppress warnings relative to null analysis（为了抑制与null分析相关的警告） |
| rawtypes                 | to suppress warnings relative to un-specific types when using generics on class params（在类params上使用泛型时，要抑制相对于非特异性类型的警告） |
| restriction              | to suppress warnings relative to usage of discouraged or forbidden references（禁止使用警告或禁止引用的警告） |
| serial                   | to suppress warnings relative to missing serialVersionUID field for a serializable class（为了一个可串行化的类，为了抑制相对于缺失的serialVersionUID字段的警告） |
| static-access            | o suppress warnings relative to incorrect static access（o抑制与不正确的静态访问相关的警告） |
| synthetic-access         | to suppress warnings relative to unoptimized access from inner classes（相对于内部类的未优化访问，来抑制警告） |
| unchecked                | to suppress warnings relative to unchecked operations（相对于不受约束的操作，抑制警告） |
| unqualified-field-access | to suppress warnings relative to field access unqualified（为了抑制与现场访问相关的警告） |
| unused                   | to suppress warnings relative to unused code（抑制没有使用过代码的警告） |



**五、Java Lint 选项**　　　　　　　　　　　　　　　　　　　　　　　　　　　

1. **lint的含义**

   用于在编译程序的过程中，进行更细节的额外检查。

2. **javac 的标准选项和非标准选项**

　　**标准选项：是指当前版本和未来版本中都支持的选项，如 -cp 和 -d 等。**

   ***非标准选项：*是指当前版本支持，但未来不一定支持的选项。通过 javac -X 查看当前版本支持的非标准选项。**

![image-20210309111348063](https://gitee.com/wanxianbo/pic-bed/raw/master/img/2021/03/20210309111404.png)

 ***3.* *查看警告信息***

  **默认情况下执行 javac 仅仅显示警告的扼要信息，也不过阻止编译过程。若想查看警告的详细信息，则需要执行 javac -Xlint:keyword 来编译源码了**