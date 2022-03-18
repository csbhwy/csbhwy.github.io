@[toc]
##### 1. public int length ()
```java
String s = "helloworld";
int length = s.lenght();
//获取字符串长度，lenght的结果为10
```
##### 2. public String concat (String str)

```java
String s1 = "helloworld";
String s2 = s1.concat("，everyone");
//String concat:将指定的字符串连接到该字符串的末尾，s2为"helloworld, everyone"
```
##### 3. public char charAt (int index)
```java
String s = "helloworld";
char a = s.charAt(0);
char b = s.charAt(1)
//char charAt：获取指定索引处的字符，a = ‘h’;b = ‘e’
```
##### 4. public int indexOf (String str)
```java
String s = "helloworld";
int a = s.indexOf("l");
int b = s.indexOf("hwm");
// 获取子字符串第一次出现在该字符串内的索引，没有返回-1,a = 2;b = -1
```
##### 5. public String substring (int beginIndex)
```java
String s = "helloworld";
String a = s.substring(0);
String b = s.substring(5);
// 从beginIndex开始截取字符串到字符串结尾,a = "helloworld";b = "world"
```
##### 6. public String substring (int beginIndex, int endIndex)
```java
String s = "helloworld";
String a = s.substring(0, s.length());
String b = s.substring(3,8);
// 从beginIndex到endIndex截取字符串。含beginIndex，不含endIndex。a = "helloworld",b = "lowor"
```
##### 7. public char[] toCharArray ()
```java
String s = "helloworld!";
char[] a = s.toCharArray();
//char[] toCharArray:把字符串转换为字符数组
```
##### 8. public byte[] getBytes ()
```java
String s = "HelloWorld!";
byte[] bytes = s.getBytes();
//使用平台的默认字符集将该String编码转换为新的字节数组
```
##### 9. public String[] split(String regex)
```java
String s = "aa|bb|cc";
String[] strArray = s.split("\\|"); for(int i = 0; i < strArray.length; i++){
   System.out.print(strArray[i]);
}
//将此字符串按照给定的regex（规则）拆分为字符串数组(特殊符号需要用 反斜杠 \ 转义，在Java要用两个反斜杠 \\)
```
##### 10. boolean contains(CharSequence s)
```java
String s = "djlfdjksdlka";
boolean str = s.contains("g");
//判断字符串中是否包含指定字符
```
##### 11. public String replace (CharSequence target, CharSequence replacement)
```java
String str = "itcast itheima";
String replace = str.replace("it","IT");
//将与target匹配的字符串使用replacement字符串替换。
```
##### 12. public String toLowerCase()
```java
String s = "HelloWorld!";
String a = s.toLowerCase();
//将字符串中的所有字符全部转换成小写，而非字母的字符不受影响,a = "helloworld!"
```
##### 13. public String toUpperCase()
```java
String s = "HelloWorld!";
String a = s.toUpperCase();
//将字符串中的所有字符全部转换成大写，而非字母的字符不受影响,a = "HELLOWORLD!"
```
##### 14. public String trim()
```java
String s = "    HelloWorld!  ";
String a = s.trim();
//用于删除字符串的头尾空白符,a = "HelloWorld!"
```
##### 15. public boolean endsWith(String suffix)
```java
String s = "HelloWorld";
boolean a = s.endsWith("World");
//如果参数表示的字符序列是此对象表示的字符序列的后缀，则返回 true；否则返回 false
```
##### 16. public boolean isEmpty()
```java
//判断字符串是否为空，已分配内存空间，但未初始化，要区别于null和“”空字符串
```
##### 17. public boolean startsWith(String prefix)
```java
String s = "HelloWorld";
boolean a = s.startsWith("start");
//startsWith() 方法用于检测字符串是否以指定的前缀开始, a = true
```
##### 18. public boolean startsWith(String prefix, int toffset)
```java
String s = "HelloWorld";
boolean a = s.startsWith("Wor", 5);
//用于检测从toffset偏移量位置开始，是否接下来的字符串为prefix, a = true
```
##### 19. public boolean equalsIgnoreCase(String anotherString)
```java
//忽略大小写比较两个字符串
```



