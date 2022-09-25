---
title: 'java基础之一文件操作'
date: 2021-05-08 14:57:30
tags: [java]
published: true
hideInList: false
feature: /post-images/FNm9kw7nn.png
isTop: false
---
![](https://tinaxiawuhao.github.io/post-images/1620370686314.png)
### 一、抽象类：

字节流： `InputStream`（读入流） `OutputStream`(写出流）
字符流：` Reader`（字符 读入流） `Writer` （字符写出流）

### 字节流读取字符数据的问题 :

 汉字等字符,往往有多个字节组成,那么到底哪几个字节应该组成一个汉字呢? 字节流不知道.如果我们自己用字节流读取字节数据,然后手动自己转字符,就有可能对规则不了解.从而造成错误. Java中的字符流恰好就可以帮助我们解决这个问题.

Java提供的字符流的底层，它自己会根据读取到的字节数据的特点，然后自动的帮助我们把字节转成字符，最后我们就会自然的得到想要的字符数据。

**字符流怎么解决这个问题呢?**

​    **字符流 =** **字节流 +** **编码表**

编码表其实里面定义了字节与字符的转换规则.

字符流在读取数据的时候,底层还是字节流在读取数据,字符流会自动根据编码规则把字节数据转为字符数据,就不会造成错误!

### 编码表的介绍 :

**说明 :**  计算机中保存的数据都是二进制（1010100101001），我们要把生活中的数据保存到计算机中，需要把生活中的数据转成二进制数据，然后才能让计算机来识别生活中的数据，进而对这些数据进行处理。 "补充 :" 要把生活中的数据转成计算机能够识别的数据，就需要定义一个固定的转换关系： 老美他们为了把自己的生活中的数据保存到计算机中，他们发明了一张表，然后在这张表中规定了生活中所有的字符和二进制之间的对应关系：老美的文字和计算机中的二进制的对应关系表：ASCII。

 **ASCII码表：** 它采用的是一个字节表示所有的文字（标点符号，英文字母，其他的特殊符号，数字等）。 

|生活中的字符   | 十进制 |   二进制 |
|---|---|---|
|A|65| 01000001 |
|B|66|  01000010 |
 欧洲也定义一张拉丁码表：ISO-8859-1 这个表码表兼容ASCII码表。也采用的一个字节表示字符数据。

 **ASCII：**  一个字节表示一个字符： `0xxx xxxx`  规定二进制的最高位是0，其他的7位表示某个字符的编码值。 

**ISO-8859-1：**  一个字节表示一个字符：把整个字节表示字符，`xxxx xxxx` 全部用来表示字符数据 

**中国的编码表：** GBK编码表采用2个字节表示一个汉字。
**GB2312 ：** 它识别六七千的文字。兼容ASCII编码表。              

**GBK：** 识别两万多字符。目前主流的编码表.              

**GB18030：** 识别更多。

世界计算机协会也制定了一个张国际通用的编码表：

​       **Unicode编码表：** 它也采用2个字节表示一个字符。       

Unicode编码表升级：**UTF-8**。       

​	 **UTF-8：**  它对字符的编码规律可以使用一个字节表示的字符就使用一个字节。 可以使用两个字节表示的字符就使用两个字节。 可以使用三个字节表示的字符就使用三个字节（基本汉字都使用三个字节）。 

可以识别汉字的编码表：      ` GB2312`、`GBK`、`Unicode`、`UTF-8`

### 二、文件操作流
字节流： `FileInputStream` ，`FileOutputStream`
字符流： `FileReader`, `FileWriter`

```java
File file = new File("D:/test/testIO.java");
InputStream in = new FileInputStream(file);
InputStream in = new FileInputStream("D:/test/testIO.java");
OutputStream out = new FileOutputStream(file);
OutputStream out = new FileOutputStream("D:/test/testIO.java");
```

```java
Reader reader = new FileReader("demo01.txt");
FileWriter writer = new FileWriter("demo02.txt");
Writer writer = new OutputStreamWriter(new FileOutputStream("demo03_utf8.txt"), "UTF-8");
Reader reader = new InputStreamReader(new FileInputStream("demo03_utf8.txt"), "UTF-8");

```

```java
//1.指定要读 的文件目录及名称
File file =new File("文件路径");
//2.创建文件读入流对象
FileInputStream fis =new FileInputStream(file);
//3.定义结束标志,可用字节数组读取
int i =0 ;
while((i = fis.read())!=-1){ 
    //i 就是从文件中读取的字节，读完后返回-1
}
//4.关闭流
fis.close();
//5.处理异常
```



```java
//1.指定要写到的文件目录及名称
File file =new File("文件路径");
//2.创建文件读入流对象
FileOutputStream fos =new FileOutputStream(file);
//3.定义结束标志
fos.write(要写出的字节或者字节数组);
//4.刷新和关闭流
fos.flush();
fos.close();
//5.处理异常
```

###  三、缓冲流：
​    字节缓冲流：` BufferedInputStream`,`BufferedOutputStream`
​    字符缓冲流：`BufferedReader` ,`BufferedWriter`
​    缓冲流是对流的操作的功能的加强，提高了数据的读写效率。既然缓冲流是对流的功能和读写效率的加强和提高，所以在创建缓冲流的对象时应该要传入要加强的流对象。

```java
//保存其参数，即输入流 in，以便将来使用。创建一个内部缓冲区数组并将
//其存储在 buf 中,该buf的大小默认为8192。
public BufferedInputStream(InputStream in);

 
//创建具有指定缓冲区大小的 BufferedInputStream 并保存其参数，
//即输入流 in，以便将来使用。创建一个长度为 size 的内部缓冲区数组并
//将其存储在 buf 中。
public BufferedInputStream(InputStream in,int size);


//创建一个新的缓冲输出流，以将数据写入指定的底层输出流。
public BufferedOutputStream(OutputStream out);

 
//创建一个新的缓冲输出流，以将具有指定缓冲区大小的数据写入指定的底层输出流。
public BufferedOutputStream(OutputStream out,int size);
```

```java
BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(new FileOutputStream("demo08_utf8.txt"), "UTF-8"));

BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("demo08_utf8.txt"), "UTF-8"));
```

```java
//1.指定要读 的文件目录及名称
File file =new File("文件路径");
//2.创建文件读入流对象
FileInputStream fis =new FileInputStream(file);
//3.创建缓冲流对象加强fis功能
BufferedInputStream bis =new BufferedInputStream(fis); 
//4.定义结束标志,可用字节数组读取
int i =0 ;
while((i = bis.read())!=-1){ 
    //i 就是从文件中读取的字节，读完后返回-1
}
//5.关闭流
bis.close();
//6.处理异常
```

```java
//1.指定要写到的文件目录及名称
File file =new File("文件路径");
//2.创建文件读入流对象
FileOutputStream fos =new FileOutputStream(file);
//3.创建缓冲流对象加强fos功能
BufferedOutputStream bos=new BufferedOutputStream(fos);
//4.向流中写入数据
bos.write(要写出的字节或者字节数组);
//5.刷新和关闭流
bos.flush();
bos.close();
//6.处理异常
```

由以上看出流的操作基本相同，此流与文件流操作是几乎一样的只是将文件流作为参数传入缓冲流的构造方法中堆文件流读写文件的功能进行加强
注1：在字符读入缓冲流`BufferedReade`r 中还提供了读一行的方法 `readLine() `可以读取一行文本
在字符写出缓冲流`BufferedWriter` 中还提供了写入一个行分隔符的方法`writeLine()`,用于写出时换行
注2：此处用到的是设计模式中的装饰模式

```java
//装饰的对象只要是抽象类的子类即可
BufferedInputStream bis =new BufferedInputStream(new FileInputStream(new File("文件路径")));   
```

### 四、转换流：
这类流是用于将字符转换为字节输入输出，用于操作字符文件，属于字符流的子类，所以后缀为`reader`，`writer`；前缀`inputstream`，`outputstream`；

注 ：要传入字节流作为参赛
`InputStreamReade`r: 字符转换输出流
`OutputStreamWriter`：字符转换输入流

```java
需求：读取键盘输入的一行文本,再将输入的写到本地磁盘上
//1.获取键盘输入的字节流对象in
InputStream in =Stream.in; 
/*2.用转换流将字节流对象转换为字符流对象，方便调用字符缓冲流的readeLine()方法*/
InputStreamReader isr =new InputStreamReader(in);
/*5.创建字符转换输出流对象osw，方便把输入的字符流转换为字节输出到本地文件。*/
OutputStreamWriter osw =new OutputStreamWriter(new FileOutputStream(new File("文件名"))); 
/*3.现在isr是字符流，可以作为参数传入字符缓冲流中*/
BufferedReader br =new BufferedReader(isr);
/*4.可以调用字符缓冲流br的readLine()方法度一行输入文本*/
String line =null;
while((line =br.readLine()){
    osw.write(line);//osw是字符流对象，可以直接操作字符串
}
//刷新此缓冲的输出流，保证数据全部都能写出
osw.flush();
br.close();
osw.close();
```

### 读取数据

```java
//读取方式一 : 效率低!
int content = -1;
while ((content = reader.read()) != -1) {
    System.out.print((char)content);
}


//读取方式二 : 数组之后的无用数据都会被读取!
int len = -1;
char[] cbuf = new char[1024];
while ((len = reader.read(cbuf)) != -1) {
    System.out.println(cbuf);
}


// 读取方式三 : 将读取的数据内容转换为 `字符串`
int len = -1;
char[] cbuf = new char[1024];
while ((len = reader.read(cbuf)) != -1) {
    String str = new String(cbuf, 0, len);
    System.out.println(str);
}
```

```java
"转换流 和 操作流 之间的关系 :"
"转换流 :"
1. OutputStreamWriter = OutputStream + 任意编码表
2. InputStreamReader = InputStream + 任意编码表
"操作流 :"
1. FileWriter = OutputStreamWriter + 默认编码表 (GBK)
2. FileReader = InputStreamReader + 默认编码表(GBK)

"总结 : 一般我们都使用操作流,当需要指定编码表的时候,才会使用转换流"
```

### 拆分文件及还原

```java
public class Test {
     public static void main(String[] args) throws IOException {
           int count = judge();
           System.out.println(count);
           BufferedInputStream bis = new BufferedInputStream(new FileInputStream(
                     "somethings\\test.flv"));
           incision(count, bis, 6);
           BufferedOutputStream bos = new BufferedOutputStream(
                     new FileOutputStream("somethings\\jzc.flv"));
           joint(bos,6);
           bos.close();
           bis.close();
     }
     public static void joint( BufferedOutputStream bos,int n)
                throws FileNotFoundException, IOException {
           for (int i = 1; i <= n; i++) {
                BufferedInputStream bis = new BufferedInputStream(
                           new FileInputStream("somethings\\jzc" + i + ".flv"));
                byte[] bys = new byte[1024];
                int len = -1;
                while ((len = bis.read(bys)) != -1) {
                     bos.write(bys, 0, len);
                }
                bis.close();
           }
           bos.close();
           System.out.println("拼接完了");
     }
     public static void incision(int count, BufferedInputStream bis, int n)
                throws FileNotFoundException, IOException {
           byte[] bys = new byte[1024];
           int len = -1;
           
           for (int i = 1; i <= n; i++) {
                int count2 = 0;
                BufferedOutputStream bos = new BufferedOutputStream(
                           new FileOutputStream("somethings\\jzc" + i + ".flv"));
                while ((len = bis.read(bys)) != -1) {
                     count2 += 1024;
                     System.out.println(count2);
                     if (count2 < (count / n)+1024&&count2 > (count / n)-1024) {
                           break;
                     } else {
                           bos.write(bys, 0, len);
                     }
                }
                bos.close();
           }
           System.out.println("切割完了");
     }
     public static int judge() throws FileNotFoundException, IOException {
           BufferedInputStream bis = new BufferedInputStream(new FileInputStream(
                     "somethings\\test.flv"));
           byte[] bys = new byte[1024];
           int len = -1;
           int count = 0;
           while ((len = bis.read(bys)) != -1) {
                count += 1024;
           }
           bis.close();
           return count;
     }
}
```