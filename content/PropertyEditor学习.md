## 简介

​	在SpringMVC架构中，从request或默认数据中提取意向数据后，需要将其转换成request-method-parameter所需的类型。这个过程中，WebDataBinder应用了java.beans.PropertyEditor，实现数据转型。自定义情况下，应当继承java.beans.PropertyEditorSupport。

​	不过由于PropertyEditor本来是支持GUI的，因此，有部分api不应实现，比如paintValue、getCustomEditor。其自述有三种简单的风格，这里不写了。

## 接口介绍

```java
// 被编辑数据的getter和setter，基本数据类型需要包装。
void setValue(Object value);
Object getValue();
```

```java
// GUI应用的专属类，needn't care 
boolean isPaintable();
void paintValue(java.awt.Graphics gfx, java.awt.Rectangle box);
```

```java
// 使用貌似丰富，but Initialization is keyworkd。
String getJavaInitializationString();
```

```java
// human editable string，也就是文本内容。springMVC就是将文本表达的意图，以对象的形式再现。
// 从spring中提供的实现来看，getAsText并不是setAsText调用时传入参数，而是text转化后的对象，从自己信息提取的再现。就像模拟信号->数字信号->模拟型号就会失真。不过我们意图就是获取数字信号，丢失的数据并不需要则无关紧要。
// 以数据源种类来看，纯本文应当用此处方法，非纯文本用getValue()和setValue(.)更常用些。Spring都是用此处风法调用getValue()、setValue(.)。
String getAsText();
void setAsText(String text) throws java.lang.IllegalArgumentException;
```

```java
// 本属性值代表的标签。如果实现tags，则getAsText()和setAsText(.)则必须实现。看注释，说是这两个方法用来设置比标签，不过如此不是只有单标签吗？那倒是value->多个tags？
String[] getTags();
```

```java
// 显然，needn't care
java.awt.Component getCustomEditor();
boolean supportsCustomEditor();
```

```java
// value change的事件监听机制涉及范畴，这里无需关心。
void addPropertyChangeListener(PropertyChangeListener listener);
void removePropertyChangeListener(PropertyChangeListener listener);
```



## PropertyEditorSupport

```java
// value和source的区别仅就当前支持类，尚不明确
// 被监听的值。
private Object value;
// 数据源，构造时期给定的。未给与，source=this。
private Object source;
// 无需关心
private java.util.Vector<PropertyChangeListener> listeners;
```

```java
public void setValue(Object value) {
    this.value = value;
    // value值改变，触发listener。
    firePropertyChange();
}
```

```java
public Object getValue() {
    return value;
}
```

注：观赏Spring中的PropertyEditorSupport的继承类，发现source没有被用到。value的注入是`setAsText`将text转化成意向对象，setValue存储进去的。



## CustomMapEditor

​	学习一下Spring的其中一个Editor。以其为样式再去学其他的。

```java
public CustomMapEditor(Class<? extends Map> mapType, boolean nullAsEmptyMap) {
   Assert.notNull(mapType, "Map type is required");
   if (!Map.class.isAssignableFrom(mapType)) {
      throw new IllegalArgumentException(
            "Map type [" + mapType.getName() + "] does not implement [java.util.Map]");
   }
   // 给出的Map类型，传入value必定要求实在这个实现路径下，不是的话就必定转存。
   this.mapType = mapType;
   // 是否在setValue(null)的时候将null->空Map实例 存起来
   this.nullAsEmptyMap = nullAsEmptyMap;
}
```

```java
// 显然，此PropertyEditor类并不适用这两个api，至少要到子类，将text视为类全路径处理才行。
public void setAsText(String text) throws IllegalArgumentException {
   // 如此传入要抛出异常的，除非text==null
   setValue(text);
}
@Override
@Nullable
public String getAsText() {
   return null;
}
```

```java
public void setValue(@Nullable Object value) {
   // null -> 空map
   if (value == null && this.nullAsEmptyMap) {
      super.setValue(createMap(this.mapType, 0));
   }
   // 提供null 或 提供非null且不要求用新map转存
   else if (value == null || (this.mapType.isInstance(value) && !alwaysCreateNewMap())) {
      // Use the source value as-is, as it matches the target type.
      super.setValue(value);
   }
   // 此时要求转存（总是要求转存的需求，或者就是mapType不是在value的继承路径上，不符合类型要求）
   else if (value instanceof Map) {
      // Convert Map elements.
      Map<?, ?> source = (Map<?, ?>) value;
      Map<Object, Object> target = createMap(this.mapType, source.size());
      // 这里对key和value提供了两个hook接口。从这里看，转存的意义在于k-v的转变存储。
      source.forEach((key, val) -> target.put(convertKey(key), convertValue(val)));
      super.setValue(target);
   }
   // 非map类型，抛出异常
   else {
      throw new IllegalArgumentException("Value cannot be converted to Map: " + value);
   }
}
```

```java
protected Map<Object, Object> createMap(Class<? extends Map> mapType, int initialCapacity) {
   // 从以下判断中看，若指定map有序，请指定SortedMap.class传入。不要将抽象类的Class传入。
   // 非接口情况下，用反射获取无参构造函数实例化。
   if (!mapType.isInterface()) {
      try {
         return ReflectionUtils.accessibleConstructor(mapType).newInstance();
      }
      catch (Throwable ex) {
         throw new IllegalArgumentException(
               "Could not instantiate map class: " + mapType.getName(), ex);
      }
   }
   // 根据提供的Map接口的有序性质提供不同的默认map。
   else if (SortedMap.class == mapType) {
      return new TreeMap<>();
   }
   else {
      return new LinkedHashMap<>(initialCapacity);
   }
}
```

```java
// 要求进入value用新map容器进行转存期间，改变k-v。
protected Object convertKey(Object key) {
   return key;
}
protected Object convertValue(Object value) {
    return value;
}
```

## StringArrayPropertyEditor

### 字段

```java
// 默认分隔符常量
public static final String DEFAULT_SEPARATOR = ",";

// 分隔符变量
private final String separator;

// 字符串分割成子字符串数组时对每个子字符串删除其内在的子字符串，若删除后还存在再删除。
@Nullable
private final String charsToDelete;

// null->空数组 选择
private final boolean emptyArrayAsNull;

// 是否清除字符串分隔后的子字符串的两边的空白字符
private final boolean trimValues;
```

### 构造器

```java
// 都是基于四个字段进行的赋值
public StringArrayPropertyEditor(
      String separator, @Nullable String charsToDelete, boolean emptyArrayAsNull, boolean trimValues) {

   this.separator = separator;
   this.charsToDelete = charsToDelete;
   this.emptyArrayAsNull = emptyArrayAsNull;
   this.trimValues = trimValues;
}
```

### 方法

```java
public void setAsText(String text) throws IllegalArgumentException {
   // 用separator给定的分隔字符串，分隔text，并且每个单元字符串中剔除charsToDelete子字符串。
   // text==null时返回空数组；charsToDelete的删除只有在separator!=null时起作用；separator==""的意思是将text分隔成单个字符的数组。
   String[] array = StringUtils.delimitedListToStringArray(text, this.separator, this.charsToDelete);
   // 好家伙，空数组也给你用null传传进去了
   if (this.emptyArrayAsNull && array.length == 0) {
      setValue(null);
   }
   // 将每个单位字符串的两边空白字符清空。
   else {
      if (this.trimValues) {
         array = StringUtils.trimArrayElements(array);
      }
      setValue(array);
   }
}
```

```java
// 从这里看gateAsText对文本数据源的返回并不会原样返回，而是基于存储理念/结构数据value，将其以字符串的形式转存输出。
public String getAsText() {
   // 将字符串数组用分隔符间隔拼接并返回。底层value时String[]，理解时无需过多关心类型问题。
   return StringUtils.arrayToDelimitedString(ObjectUtils.toObjectArray(getValue()), this.separator);
}
```

## ByteArrayPropertyEditor

```java
// 简答的转文本存储，据说高版本的jdk将String的底层存储结构改变了，无妄之灾。
@Override
public void setAsText(@Nullable String text) {
   setValue(text != null ? text.getBytes() : null);
}

@Override
public String getAsText() {
   byte[] value = (byte[]) getValue();
   return (value != null ? new String(value) : "");
}
```

## CharacterEditor

### 字段

```java
// unicode的转义字符前置识别
private static final String UNICODE_PREFIX = "\\u";

/**
 * The length of a Unicode character sequence.
 */
// 一个unicode字符长度————或者说一个代码单元，有些码点需要两个代码单元表示。第十七个片区？
private static final int UNICODE_LENGTH = 6;

// 是否将空null存到value？
private final boolean allowEmpty;
```

### 方法

```java
// 看来这里处理的形式只有三种，null或""(在allowEmpty=true时可行)；\uXXXX的形式传入单个代码单元；字符的形式传入码点。
public void setAsText(@Nullable String text) throws IllegalArgumentException {
   // null或""都是用null存储的，字符串就是这点麻烦空和无意义的边界都不怎么一样。
   if (this.allowEmpty && !StringUtils.hasLength(text)) {
      // Treat empty String as null value.
      setValue(null);
   }
   // 非法null，直接抛出异常
   else if (text == null) {
      throw new IllegalArgumentException("null String cannot be converted to char type");
   }
   // 看是否text是单纯的单个unicode字符
   else if (isUnicodeCharacterSequence(text)) {
      // 以十六进制看其数字字符，并将前置\u去掉，用char的包装类存储起来了。
      setAsUnicode(text);
   }
   // 看来这里是字符形式传入的处理
   else if (text.length() == 1) {
      setValue(text.charAt(0));
   }
   // 其他的抛出异常了
   else {
      throw new IllegalArgumentException("String [" + text + "] with length " +
            text.length() + " cannot be converted to char type: neither Unicode nor single character");
   }
}
```

```java
// 将底层Character类型值value输出。
public String getAsText() {
   Object value = getValue();
   return (value != null ? value.toString() : "");
}
```

## CharArrayPropertyEditor

```java
// 简单了些，有String的api支持。
@Override
public void setAsText(@Nullable String text) {
   setValue(text != null ? text.toCharArray() : null);
}

@Override
public String getAsText() {
   char[] value = (char[]) getValue();
   return (value != null ? new String(value) : "");
}
```

## CharsetEditor - 不会标记

```java
// 就Charset的类注释所述，其是16bit Unicode单元 和 byte数组之间的映射。
// Charset - 字符集。饶了自己吧，这个暂时不学。
public void setAsText(String text) throws IllegalArgumentException {
   if (StringUtils.hasText(text)) {
      setValue(Charset.forName(text.trim()));
   }
   else {
      setValue(null);
   }
}

@Override
public String getAsText() {
   Charset value = (Charset) getValue();
   return (value != null ? value.name() : "");
}
```

## ClassArrayEditor

### 字段

```java
private final ClassLoader classLoader;
```

### 方法

```java
public void setAsText(String text) throws IllegalArgumentException {
   if (StringUtils.hasText(text)) {
      // 用共识","分隔符
      String[] classNames = StringUtils.commaDelimitedListToStringArray(text);
      Class<?>[] classes = new Class<?>[classNames.length];
      for (int i = 0; i < classNames.length; i++) {
         // 去除两边空白字符
         String className = classNames[i].trim();
         // classLoader + className（全路径） -> Class
         classes[i] = ClassUtils.resolveClassName(className, this.classLoader);
      }
      setValue(classes);
   }
   else {
      setValue(null);
   }
}
```

```java
// Class数组 -> 全路径名数组 -> 逗号分隔 
public String getAsText() {
   Class<?>[] classes = (Class[]) getValue();
   if (ObjectUtils.isEmpty(classes)) {
      return "";
   }
   StringJoiner sj = new StringJoiner(",");
   for (Class<?> klass : classes) {
      sj.add(ClassUtils.getQualifiedName(klass));
   }
   return sj.toString();
}
```

## ClassEditor

​	类同于ClassArrayEditor，不过这里setAsText传入的text是单个类的全路径。

## CurrencyEditor

```java
//  Currency，又是一个陌生的类，就其类注释所述，是关于货币的。
@Override
public void setAsText(String text) throws IllegalArgumentException {
   if (StringUtils.hasText(text)) {
      text = text.trim();
   }
   setValue(Currency.getInstance(text));
}

@Override
public String getAsText() {
   Currency value = (Currency) getValue();
   return (value != null ? value.getCurrencyCode() : "");
}
```

## CustomBooleanEditor

### 字段

```java
// 布尔值与字符串的映射关系
public static final String VALUE_TRUE = "true";
public static final String VALUE_FALSE = "false";

public static final String VALUE_ON = "on";
public static final String VALUE_OFF = "off";

public static final String VALUE_YES = "yes";
public static final String VALUE_NO = "no";

public static final String VALUE_1 = "1";
public static final String VALUE_0 = "0";

// 用于自定义字符串与布尔值的映射
@Nullable
private final String trueString;
@Nullable
private final String falseString;
// 是否允许null存储。
private final boolean allowEmpty;
```

### 构造函数

```java
public CustomBooleanEditor(@Nullable String trueString, @Nullable String falseString, boolean allowEmpty) {
   this.trueString = trueString;
   this.falseString = falseString;
   this.allowEmpty = allowEmpty;
}
```

### 方法

```java
public void setAsText(@Nullable String text) throws IllegalArgumentException {
   // 非空则移除两边空白字符
   String input = (text != null ? text.trim() : null);
   // null或""置value=null
   if (this.allowEmpty && !StringUtils.hasLength(input)) {
      // Treat empty String as null value.
      setValue(null);
   }
   // 自定义true、false 与 字符串的映射 判断
   else if (this.trueString != null && this.trueString.equalsIgnoreCase(input)) {
      setValue(Boolean.TRUE);
   }
   else if (this.falseString != null && this.falseString.equalsIgnoreCase(input)) {
      setValue(Boolean.FALSE);
   }
   // 以下就是true-false / on-off / yes-no / 1-0 向Boolean的映射
   else if (this.trueString == null &&
         (VALUE_TRUE.equalsIgnoreCase(input) || VALUE_ON.equalsIgnoreCase(input) ||
               VALUE_YES.equalsIgnoreCase(input) || VALUE_1.equals(input))) {
      setValue(Boolean.TRUE);
   }
   else if (this.falseString == null &&
         (VALUE_FALSE.equalsIgnoreCase(input) || VALUE_OFF.equalsIgnoreCase(input) ||
               VALUE_NO.equalsIgnoreCase(input) || VALUE_0.equals(input))) {
      setValue(Boolean.FALSE);
   }
   // 不能映射到布尔值，抛出异常
   else {
      throw new IllegalArgumentException("Invalid boolean value [" + text + "]");
   }
}
```

```java
// 返回自定义的true-false表达的字符串，若没有则返回true-false字符。
public String getAsText() {
   if (Boolean.TRUE.equals(getValue())) {
      return (this.trueString != null ? this.trueString : VALUE_TRUE);
   }
   else if (Boolean.FALSE.equals(getValue())) {
      return (this.falseString != null ? this.falseString : VALUE_FALSE);
   }
   else {
      return "";
   }
}
```

## CustomCollectionEditor

### 字段

```java
// 自建的类型要求。List.class -> ArrayList.class、SortedSet -> TreeSet、其他接口类型 -> LinkedHashSet。若是具体类，则反射调用无参构造函数实例化。不能给定抽象类Class。
private final Class<? extends Collection> collectionType;

// null->空集合 允许？
private final boolean nullAsEmptyCollection;
```

### 构造器

```java
public CustomCollectionEditor(Class<? extends Collection> collectionType, boolean nullAsEmptyCollection) {
   Assert.notNull(collectionType, "Collection type is required");
   if (!Collection.class.isAssignableFrom(collectionType)) {
      throw new IllegalArgumentException(
            "Collection type [" + collectionType.getName() + "] does not implement [java.util.Collection]");
   }
   this.collectionType = collectionType;
   this.nullAsEmptyCollection = nullAsEmptyCollection;
}
```

### 方法

```java
public void setAsText(String text) throws IllegalArgumentException {
   setValue(text);
}
```

```java
public void setValue(@Nullable Object value) {
   if (value == null && this.nullAsEmptyCollection) {
      super.setValue(createCollection(this.collectionType, 0));
   }
   // 这里alwaysCreateNewCollection()=false，即不要求在内部创建新集合转存。
   else if (value == null || (this.collectionType.isInstance(value) && !alwaysCreateNewCollection())) {
      // Use the source value as-is, as it matches the target type.
      super.setValue(value);
   }
   // 这里必当需要转存（这里的情况一定是collectionType不是value的继承链路上）
   // value是集合
   else if (value instanceof Collection) {
      // Convert Collection elements.
      Collection<?> source = (Collection<?>) value;
      Collection<Object> target = createCollection(this.collectionType, source.size());
      for (Object elem : source) {
         target.add(convertElement(elem));
      }
      super.setValue(target);
   }
   // value是数组，数组->列表
   else if (value.getClass().isArray()) {
      // Convert array elements to Collection elements.
      int length = Array.getLength(value);
      Collection<Object> target = createCollection(this.collectionType, length);
      for (int i = 0; i < length; i++) {
         target.add(convertElement(Array.get(value, i)));
      }
      super.setValue(target);
   }
   // 其他情况：将value视为单个元素存入新容器。
   else {
      // A plain value: convert it to a Collection with a single element.
      Collection<Object> target = createCollection(this.collectionType, 1);
      // convertElement(.)返回原样，后期可拓展。
      target.add(convertElement(value));
      super.setValue(target);
   }
}
```

```java
protected Collection<Object> createCollection(Class<? extends Collection> collectionType, int initialCapacity) {
   // 非接口，具体类类型
   if (!collectionType.isInterface()) {
      try {
         return ReflectionUtils.accessibleConstructor(collectionType).newInstance();
      }
      catch (Throwable ex) {
         throw new IllegalArgumentException(
               "Could not instantiate collection class: " + collectionType.getName(), ex);
      }
   }
    
   // List.class -> ArrayList.class
   else if (List.class == collectionType) {
      return new ArrayList<>(initialCapacity);
   }
   // SortedSet -> TreeSet
   else if (SortedSet.class == collectionType) {
      return new TreeSet<>();
   }
   // 其他接口类型 -> LinkedHashSet
   else {
      return new LinkedHashSet<>(initialCapacity);
   }
}
```

```java
protected boolean alwaysCreateNewCollection() {
   return false;
}
```

```java
protected Object convertElement(Object element) {
   return element;
}
```

```java
public String getAsText() {
   return null;
}
```

## CustomDateEditor - 不会标记

### 字段

```java
private final DateFormat dateFormat;

// 允许null注入
private final boolean allowEmpty;

// 确切的日期长度。负数表示不要求，一般指定-1
private final int exactDateLength;
```

### 构造器

```java
public CustomDateEditor(DateFormat dateFormat, boolean allowEmpty, int exactDateLength) {
   this.dateFormat = dateFormat;
   this.allowEmpty = allowEmpty;
   this.exactDateLength = exactDateLength;
}
```

### 方法

```java
// 学习这边的得先学习时间的，唉。
@Override
public void setAsText(@Nullable String text) throws IllegalArgumentException {
   // null 或 没有有意义的文本（空白字符算无意义的）
   if (this.allowEmpty && !StringUtils.hasText(text)) {
      // Treat empty String as null value.
      setValue(null);
   }
   // 指定时间文本长度，不符合则抛异常
   else if (text != null && this.exactDateLength >= 0 && text.length() != this.exactDateLength) {
      throw new IllegalArgumentException(
            "Could not parse date: it is not exactly" + this.exactDateLength + "characters long");
   }
   // 时间这东西真烦。
   else {
      try {
         // this.dateFormat.parse(text)将日期换成时间了，不过这个应该是用了当前系统的时区。
         setValue(this.dateFormat.parse(text));
      }
      catch (ParseException ex) {
         throw new IllegalArgumentException("Could not parse date: " + ex.getMessage(), ex);
      }
   }
}

/**
 * Format the Date as String, using the specified DateFormat.
 */
@Override
public String getAsText() {
   Date value = (Date) getValue();
   return (value != null ? this.dateFormat.format(value) : "");
}
```

## CustomNumberEditor - 不会标记

### 字段

```java
private final Class<? extends Number> numberClass;

@Nullable
private final NumberFormat numberFormat;

private final boolean allowEmpty;
```

### 构造函数

```java
public CustomNumberEditor(Class<? extends Number> numberClass,
      @Nullable NumberFormat numberFormat, boolean allowEmpty) throws IllegalArgumentException {

   if (!Number.class.isAssignableFrom(numberClass)) {
      throw new IllegalArgumentException("Property class must be a subclass of Number");
   }
   this.numberClass = numberClass;
   this.numberFormat = numberFormat;
   this.allowEmpty = allowEmpty;
}
```

### 方法

```java
public void setAsText(String text) throws IllegalArgumentException {
   if (this.allowEmpty && !StringUtils.hasText(text)) {
      // Treat empty String as null value.
      setValue(null);
   }
   
   else if (this.numberFormat != null) {
      // Use given NumberFormat for parsing text.
      setValue(NumberUtils.parseNumber(text, this.numberClass, this.numberFormat));
   }
   else {
      // 不指定numberFormat，用numberClass的类型，与相对应的类型的api来解决，不清楚与numberFormat来解析的区别。
      // Use default valueOf methods for parsing text.
      setValue(NumberUtils.parseNumber(text, this.numberClass));
   }
}
```

```java
public void setValue(@Nullable Object value) {
   if (value instanceof Number) {
      // 将value转换成numberClass类型存储。
      super.setValue(NumberUtils.convertNumberToTargetClass((Number) value, this.numberClass));
   }
   else {
      super.setValue(value);
   }
}
```

```java
public String getAsText() {
   Object value = getValue();
   if (value == null) {
      return "";
   }
   if (this.numberFormat != null) {
      // Use NumberFormat for rendering value.
      return this.numberFormat.format(value);
   }
   else {
      // Use toString method for rendering value.
      return value.toString();
   }
}
```

## ResourceEditor

唉，还有一些就不往下学了，还是掌握使用场景重要些。









## 结尾





