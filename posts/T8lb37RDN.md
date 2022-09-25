---
title: 'easyExcel'
date: 2022-03-10 18:50:56
tags: [excel]
published: true
hideInList: false
feature: /post-images/T8lb37RDN.png
isTop: false
---


### 读Excel

#### 注解

使用注解很简单，只要在对应的实体类上面加上注解即可。

##### `ExcelProperty`

用于匹配excel和实体类的匹配,参数如下：

| 名称      | 默认值            | 描述                                                         |
| --------- | ----------------- | ------------------------------------------------------------ |
| value     | 空                | 用于匹配excel中的头，必须全匹配,如果有多行头，会匹配最后一行头 |
| order     | Integer.MAX_VALUE | 优先级高于`value`，会根据`order`的顺序来匹配实体和excel中数据的顺序 |
| index     | -1                | 优先级高于`value`和`order`，会根据`index`直接指定到excel中具体的哪一列 |
| converter | 自动选择          | 指定当前字段用什么转换器，默认会自动选择。读的情况下只要实现`com.alibaba.excel.converters.Converter#convertToJavaData(com.alibaba.excel.converters.ReadConverterContext<?>)` 方法即可 |

##### `ExcelIgnore`

默认所有字段都会和excel去匹配，加了这个注解会忽略该字段

##### `ExcelIgnoreUnannotated`

默认不加`ExcelProperty` 的注解的都会参与读写，加了不会参与

##### `DateTimeFormat`

日期转换，用`String`去接收excel日期格式的数据会调用这个注解,参数如下：

| 名称             | 默认值   | 描述                                                         |
| ---------------- | -------- | ------------------------------------------------------------ |
| value            | 空       | 参照`java.text.SimpleDateFormat`书写即可                     |
| use1904windowing | 自动选择 | excel中时间是存储1900年起的一个双精度浮点数，但是有时候默认开始日期是1904，所以设置这个值改成默认1904年开始 |

##### `NumberFormat`

数字转换，用`String`去接收excel数字格式的数据会调用这个注解。

| 名称         | 默认值               | 描述                                  |
| ------------ | -------------------- | ------------------------------------- |
| value        | 空                   | 参照`java.text.DecimalFormat`书写即可 |
| roundingMode | RoundingMode.HALF_UP | 格式化的时候设置舍入模式              |

#### 参数

##### 概念介绍

- `ReadWorkbook` 可以理解成一个excel
- `ReadSheet` 理解成一个excel里面的一个表单

##### 通用参数

`ReadWorkbook`,`ReadSheet` 都会有的参数，如果为空，默认使用上级。

| 名称                | 默认值 | 描述                                                         |
| ------------------- | ------ | ------------------------------------------------------------ |
| converter           | 空     | 默认加载了很多转换器，这里可以加入不支持的字段               |
| readListener        | 空     | 可以注册多个监听器，读取excel的时候会不断的回调监听器中的方法 |
| headRowNumber       | 1      | excel中头的行数，默认1行                                     |
| head                | 空     | 与`clazz`二选一。读取文件头对应的列表，会根据列表匹配数据，建议使用class |
| clazz               | 空     | 与`head`二选一。读取文件的头对应的class，也可以使用注解。如果两个都不指定，则会读取全部数据 |
| autoTrim            | true   | 会对头、读取数据等进行自动trim                               |
| use1904windowing    | false  | excel中时间是存储1900年起的一个双精度浮点数，但是有时候默认开始日期是1904，所以设置这个值改成默认1904年开始 |
| useScientificFormat | false  | 数字转文本的时候在较大的数值的是否是否采用科学计数法         |

##### ReadWorkbook

设置方法如下，找不到参数的看下通用参数里面是否存在。

```java
 EasyExcel.read(fileName, DemoData.class, new DemoDataListener())
 // 在 read 方法之后， 在 sheet方法之前都是设置ReadWorkbook的参数
 .sheet()
 .doRead();
```

| 名称                     | 默认值                  | 描述                                                         |
| ------------------------ | ----------------------- | ------------------------------------------------------------ |
| excelType                | 空                      | 当前excel的类型,支持XLS、XLSX、CSV                           |
| inputStream              | 空                      | 与`file`二选一。读取文件的流，如果接收到的是流就只用，不用流建议使用`file`参数。因为使用了`inputStream` easyexcel会帮忙创建临时文件，最终还是`file` |
| file                     | 空                      | 与`inputStream`二选一。读取文件的文件。                      |
| mandatoryUseInputStream  | false                   | 强制使用 `inputStream` 来创建对象，性能会变差，但是不会创建临文件。 |
| charset                  | Charset#defaultCharset  | 只有csv文件有用，读取文件的时候使用的编码                    |
| autoCloseStream          | true                    | 自动关闭读取的流。                                           |
| readCache                | 空                      | 默认小于5M用 内存，超过5M会使用 `EhCache`,这里不建议使用这个参数。 |
| readCacheSelector        | SimpleReadCacheSelector | 用于选择什么时候用内存去存储临时数据，什么时候用磁盘存储临时数据 |
| ignoreEmptyRow           | true                    | 忽略空的行                                                   |
| password                 | 空                      | 读取文件的密码                                               |
| xlsxSAXParserFactoryName | 空                      | 指定sax读取使用的class的名称，例如：`com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl` |
| useDefaultListener       | true                    | `@since 2.1.4` 默认会加入`ModelBuildEventListener` 来帮忙转换成传入`class`的对象，设置成`false`后将不会协助转换对象，自定义的监听器会接收到`Map<Integer,CellData>`对象，如果还想继续接听到`class`对象，请调用`readListener`方法，加入自定义的`beforeListener`、 `ModelBuildEventListener`、 自定义的`afterListener`即可。 |
| extraReadSet             | 空                      | 额外需要读取内容的set，默认不读取这些数据                    |

##### ReadSheet

设置方法如下，找不到参数的看下通用参数里面是否存在。

```java
 EasyExcel.read(fileName, DemoData.class, new DemoDataListener())
 .sheet()
  // 在 sheet 方法之后， 在 doRead方法之前都是设置ReadSheet的参数
 .doRead();
```

| 名称      | 默认值 | 描述                                                 |
| --------- | ------ | ---------------------------------------------------- |
| sheetNo   | 0      | 需要读取Sheet的编码，建议使用这个来指定读取哪个Sheet |
| sheetName | 空     | 根据名字去匹配Sheet                                  |

### 写Excel

#### 注解

使用注解很简单，只要在对应的实体类上面加上注解即可。

##### `ExcelProperty`

用于匹配excel和实体类的匹配,参数如下：

| 名称      | 默认值            | 描述                                                         |
| --------- | ----------------- | ------------------------------------------------------------ |
| value     | 空                | 用于匹配excel中的头，必须全匹配,如果有多行头，会匹配最后一行头 |
| order     | Integer.MAX_VALUE | 优先级高于`value`，会根据`order`的顺序来匹配实体和excel中数据的顺序 |
| index     | -1                | 优先级高于`value`和`order`，会根据`index`直接指定到excel中具体的哪一列 |
| converter | 自动选择          | 指定当前字段用什么转换器，默认会自动选择。写的情况下只要实现`com.alibaba.excel.converters.Converter#convertToExcelData(com.alibaba.excel.converters.WriteConverterContext<T>)` 方法即可 |

##### `ExcelIgnore`

默认所有字段都会和excel去匹配，加了这个注解会忽略该字段

##### `ExcelIgnoreUnannotated`

默认不加`ExcelProperty` 的注解的都会参与读写，加了不会参与

##### `DateTimeFormat`

日期转换，用`String`去接收excel日期格式的数据会调用这个注解,参数如下：

| 名称             | 默认值   | 描述                                                         |
| ---------------- | -------- | ------------------------------------------------------------ |
| value            | 空       | 参照`java.text.SimpleDateFormat`书写即可                     |
| use1904windowing | 自动选择 | excel中时间是存储1900年起的一个双精度浮点数，但是有时候默认开始日期是1904，所以设置这个值改成默认1904年开始 |

##### `NumberFormat`

数字转换，用`String`去接收excel数字格式的数据会调用这个注解。

| 名称         | 默认值               | 描述                                  |
| ------------ | -------------------- | ------------------------------------- |
| value        | 空                   | 参照`java.text.DecimalFormat`书写即可 |
| roundingMode | RoundingMode.HALF_UP | 格式化的时候设置舍入模式              |

#### 参数

##### 概念介绍

- `WriteWorkbook` 可以理解成一个excel
- `WriteSheet` 理解成一个excel里面的一个表单
- `WriteTable` 一个表单里面如果有多个实际用的表格，则可以用`WriteTable`

##### 通用参数

`WriteWorkbook`,`WriteSheet` ,`WriteTable`都会有的参数，如果为空，默认使用上级。

| 名称                    | 默认值 | 描述                                                         |
| ----------------------- | ------ | ------------------------------------------------------------ |
| converter               | 空     | 默认加载了很多转换器，这里可以加入不支持的字段               |
| writeHandler            | 空     | 写的处理器。可以实现`WorkbookWriteHandler`,`SheetWriteHandler`,`RowWriteHandler`,`CellWriteHandler`，在写入excel的不同阶段会调用 |
| relativeHeadRowIndex    | 0      | 写入到excel和上面空开几行                                    |
| head                    | 空     | 与`clazz`二选一。读取文件头对应的列表，会根据列表匹配数据，建议使用class |
| clazz                   | 空     | 与`head`二选一。读取文件的头对应的class，也可以使用注解。如果两个都不指定，则会读取全部数据 |
| autoTrim                | true   | 会对头、读取数据等进行自动trim                               |
| use1904windowing        | false  | excel中时间是存储1900年起的一个双精度浮点数，但是有时候默认开始日期是1904，所以设置这个值改成默认1904年开始 |
| useScientificFormat     | false  | 数字转文本的时候在较大的数值的是否是否采用科学计数法         |
| needHead                | true   | 是否需要写入头到excel                                        |
| useDefaultStyle         | true   | 是否使用默认的样式                                           |
| automaticMergeHead      | true   | 自动合并头，头中相同的字段上下左右都会去尝试匹配             |
| excludeColumnIndexes    | 空     | 需要排除对象中的index的数据                                  |
| excludeColumnFieldNames | 空     | 需要排除对象中的字段的数据                                   |
| includeColumnIndexes    | 空     | 只要导出对象中的index的数据                                  |
| includeColumnFieldNames | 空     | 只要导出对象中的字段的数据                                   |

##### WriteWorkbook


设置方法如下，找不到参数的看下通用参数里面是否存在。

```java
 EasyExcel.write(fileName, DemoData.class)
            // 在 write 方法之后， 在 sheet方法之前都是设置WriteWorkbook的参数
            .sheet("模板")
            .doWrite(() -> {
                // 分页查询数据
                return data();
            });
```

| 名称                  | 默认值                 | 描述                                                         |
| --------------------- | ---------------------- | ------------------------------------------------------------ |
| excelType             | 空                     | 当前excel的类型,支持XLS、XLSX、CSV                           |
| outputStream          | 空                     | 与`file`二选一。写入文件的流                                 |
| file                  | 空                     | 与`outputStream`二选一。写入的文件                           |
| templateInputStream   | 空                     | 模板的文件流                                                 |
| templateFile          | 空                     | 模板文件                                                     |
| charset               | Charset#defaultCharset | 只有csv文件有用，写入文件的时候使用的编码                    |
| autoCloseStream       | true                   | 自动关闭写入的流。                                           |
| password              | 空                     | 读取文件的密码                                               |
| inMemory              | false                  | 是否在内存处理，默认会生成临时文件以节约内存。内存模式效率会更好，但是容易OOM |
| writeExcelOnException | false                  | 写入过程中抛出异常了，是否尝试把数据写入到excel              |

##### WriteSheet

设置方法如下，找不到参数的看下通用参数里面是否存在。

```java
 EasyExcel.write(fileName, DemoData.class)
            .sheet("模板")
             // 在 sheet 方法之后， 在 doWrite方法之前都是设置WriteSheet的参数
            .doWrite(() -> {
                // 分页查询数据
                return data();
            });
```

| 名称      | 默认值 | 描述                               |
| --------- | ------ | ---------------------------------- |
| sheetNo   | 0      | 需要写入的编码                     |
| sheetName | 空     | 需要些的Sheet名称，默认同`sheetNo` |

##### WriteTable


设置方法如下，找不到参数的看下通用参数里面是否存在。

```java
        EasyExcel.write(fileName, DemoData.class)
            .sheet("模板")
            .table()
            // 在 table 方法之后， 在 doWrite方法之前都是设置WriteTable的参数
            .doWrite(() -> {
                // 分页查询数据
                return data();
            });
```

| 名称    | 默认值 | 描述           |
| ------- | ------ | -------------- |
| tableNo | 0      | 需要写入的编码 |


```java
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>3.1.1</version>
</dependency>
```

### WebTest

```java
import java.io.IOException;
import java.net.URLEncoder;
import java.util.Date;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletResponse;

import com.alibaba.excel.EasyExcel;
import com.alibaba.excel.util.ListUtils;
import com.alibaba.excel.util.MapUtils;
import com.alibaba.fastjson.JSON;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.multipart.MultipartFile;

/**
 * web读写案例
 *
 **/
@Controller
public class WebTest {

    @Autowired
    private UploadDAO uploadDAO;

    /**
     * 文件下载（失败了会返回一个有部分数据的Excel）
     * <p>
     * 1. 创建excel对应的实体对象 参照{@link DownloadData}
     * <p>
     * 2. 设置返回的 参数
     * <p>
     * 3. 直接写，这里注意，finish的时候会自动关闭OutputStream,当然你外面再关闭流问题不大
     */
    @GetMapping("download")
    public void download(HttpServletResponse response) throws IOException {
        // 这里注意 有同学反应使用swagger 会导致各种问题，请直接用浏览器或者用postman
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setCharacterEncoding("utf-8");
        // 这里URLEncoder.encode可以防止中文乱码 当然和easyexcel没有关系
        String fileName = URLEncoder.encode("测试", "UTF-8").replaceAll("\\+", "%20");
        response.setHeader("Content-disposition", "attachment;filename*=utf-8''" + fileName + ".xlsx");

        EasyExcel.write(response.getOutputStream(), DownloadData.class).sheet("模板").doWrite(data());
    }

    /**
     * 文件下载并且失败的时候返回json（默认失败了会返回一个有部分数据的Excel）
     *
     * @since 2.1.1
     */
    @GetMapping("downloadFailedUsingJson")
    public void downloadFailedUsingJson(HttpServletResponse response) throws IOException {
        // 这里注意 有同学反应使用swagger 会导致各种问题，请直接用浏览器或者用postman
        try {
            response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            response.setCharacterEncoding("utf-8");
            // 这里URLEncoder.encode可以防止中文乱码 当然和easyexcel没有关系
            String fileName = URLEncoder.encode("测试", "UTF-8").replaceAll("\\+", "%20");
            response.setHeader("Content-disposition", "attachment;filename*=utf-8''" + fileName + ".xlsx");
            // 这里需要设置不关闭流
            EasyExcel.write(response.getOutputStream(), DownloadData.class).autoCloseStream(Boolean.FALSE).sheet("模板")
                .doWrite(data());
        } catch (Exception e) {
            // 重置response
            response.reset();
            response.setContentType("application/json");
            response.setCharacterEncoding("utf-8");
            Map<String, String> map = MapUtils.newHashMap();
            map.put("status", "failure");
            map.put("message", "下载文件失败" + e.getMessage());
            response.getWriter().println(JSON.toJSONString(map));
        }
    }

    /**
     * 文件上传
     * <p>
     * 1. 创建excel对应的实体对象 参照{@link UploadData}
     * <p>
     * 2. 由于默认一行行的读取excel，所以需要创建excel一行一行的回调监听器，参照{@link UploadDataListener}
     * <p>
     * 3. 直接读即可
     */
    @PostMapping("upload")
    @ResponseBody
    public String upload(MultipartFile file) throws IOException {
        EasyExcel.read(file.getInputStream(), UploadData.class, new UploadDataListener(uploadDAO)).sheet().doRead();
        return "success";
    }

    private List<DownloadData> data() {
        List<DownloadData> list = ListUtils.newArrayList();
        for (int i = 0; i < 10; i++) {
            DownloadData data = new DownloadData();
            data.setString("字符串" + 0);
            data.setDate(new Date());
            data.setDoubleData(0.56);
            list.add(data);
        }
        return list;
    }
}
```

### DownloadData

```java
import java.util.Date;

import com.alibaba.excel.annotation.ExcelProperty;

import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.Setter;

/**
 * 基础数据类
 *
 **/
@Getter
@Setter
@EqualsAndHashCode
public class DownloadData {
    @ExcelProperty("字符串标题")
    private String string;
    @ExcelProperty("日期标题")
    private Date date;
    @ExcelProperty("数字标题")
    private Double doubleData;
}
```

### UploadDAO

```java
import java.util.List;

import org.springframework.stereotype.Repository;

/**
 * 假设这个是你的DAO存储。当然还要这个类让spring管理，当然你不用需要存储，也不需要这个类。
 *
 **/
@Repository
public class UploadDAO {

    public void save(List<UploadData> list) {
        // 如果是mybatis,尽量别直接调用多次insert,自己写一个mapper里面新增一个方法batchInsert,所有数据一次性插入
    }
}
```

### UploadData

```java
import java.util.Date;

import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.Setter;

/**
 * 基础数据类
 *
 **/
@Getter
@Setter
@EqualsAndHashCode
public class UploadData {
    private String string;
    private Date date;
    private Double doubleData;
}
```

### UploadDataListener

```java
import java.util.List;

import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.read.listener.ReadListener;
import com.alibaba.excel.util.ListUtils;
import com.alibaba.fastjson.JSON;

import lombok.extern.slf4j.Slf4j;

/**
 * 模板的读取类
 *
 */
// 有个很重要的点 DemoDataListener 不能被spring管理，要每次读取excel都要new,然后里面用到spring可以构造方法传进去
@Slf4j
public class UploadDataListener implements ReadListener<UploadData> {
    /**
     * 每隔5条存储数据库，实际使用中可以100条，然后清理list ，方便内存回收
     */
    private static final int BATCH_COUNT = 5;
    private List<UploadData> cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
    /**
     * 假设这个是一个DAO，当然有业务逻辑这个也可以是一个service。当然如果不用存储这个对象没用。
     */
    private UploadDAO uploadDAO;

    public UploadDataListener() {
        // 这里是demo，所以随便new一个。实际使用如果到了spring,请使用下面的有参构造函数
        uploadDAO = new UploadDAO();
    }

    /**
     * 如果使用了spring,请使用这个构造方法。每次创建Listener的时候需要把spring管理的类传进来
     *
     * @param uploadDAO
     */
    public UploadDataListener(UploadDAO uploadDAO) {
        this.uploadDAO = uploadDAO;
    }

    /**
     * 这个每一条数据解析都会来调用
     *
     * @param data    one row value. Is is same as {@link AnalysisContext#readRowHolder()}
     * @param context
     */
    @Override
    public void invoke(UploadData data, AnalysisContext context) {
        log.info("解析到一条数据:{}", JSON.toJSONString(data));
        cachedDataList.add(data);
        // 达到BATCH_COUNT了，需要去存储一次数据库，防止数据几万条数据在内存，容易OOM
        if (cachedDataList.size() >= BATCH_COUNT) {
            saveData();
            // 存储完成清理 list
            cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
        }
    }

    /**
     * 所有数据解析完成了 都会来调用
     *
     * @param context
     */
    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        // 这里也要保存数据，确保最后遗留的数据也存储到数据库
        saveData();
        log.info("所有数据解析完成！");
    }

    /**
     * 加上存储数据库
     */
    private void saveData() {
        log.info("{}条数据，开始存储数据库！", cachedDataList.size());
        uploadDAO.save(cachedDataList);
        log.info("存储数据库成功！");
    }
}
```

### ExcelUtil

```java
import com.alibaba.excel.EasyExcelFactory;
import com.alibaba.excel.ExcelWriter;
import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import com.alibaba.excel.metadata.BaseRowModel;
import com.alibaba.excel.metadata.Sheet;
import lombok.Data;
import lombok.Getter;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.CollectionUtils;
import org.springframework.util.StringUtils;

import java.io.*;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Slf4j
public class ExcelUtil {

   private static Sheet initSheet;

   static {
      initSheet = new Sheet(1, 0);
      initSheet.setSheetName("sheet");
      //设置自适应宽度
      initSheet.setAutoWidth(Boolean.TRUE);
   }

   /**
    * 读取少于1000行数据
    * @param filePath 文件绝对路径
    * @return
    */
   public static List<Object> readLessThan1000Row(String filePath){
      return readLessThan1000RowBySheet(filePath,null);
   }

   /**
    * 读小于1000行数据, 带样式
    * filePath 文件绝对路径
    * initSheet ：
    *      sheetNo: sheet页码，默认为1
    *      headLineMun: 从第几行开始读取数据，默认为0, 表示从第一行开始读取
    *      clazz: 返回数据List<Object> 中Object的类名
    */
   public static List<Object> readLessThan1000RowBySheet(String filePath, Sheet sheet){
      if(!StringUtils.hasText(filePath)){
         return null;
      }

      sheet = sheet != null ? sheet : initSheet;

      InputStream fileStream = null;
      try {
         fileStream = new FileInputStream(filePath);
         return EasyExcelFactory.read(fileStream, sheet);
      } catch (FileNotFoundException e) {
         log.info("找不到文件或文件路径错误, 文件：{}", filePath);
      }finally {
         try {
            if(fileStream != null){
               fileStream.close();
            }
         } catch (IOException e) {
            log.info("excel文件读取失败, 失败原因：{}", e);
         }
      }
      return null;
   }

   /**
    * 读大于1000行数据
    * @param filePath 文件觉得路径
    * @return
    */
   public static List<Object> readMoreThan1000Row(String filePath){
      return readMoreThan1000RowBySheet(filePath,null);
   }

   /**
    * 读大于1000行数据, 带样式
    * @param filePath 文件觉得路径
    * @return
    */
   public static List<Object> readMoreThan1000RowBySheet(String filePath, Sheet sheet){
      if(!StringUtils.hasText(filePath)){
         return null;
      }

      sheet = sheet != null ? sheet : initSheet;

      InputStream fileStream = null;
      try {
         fileStream = new FileInputStream(filePath);
         ExcelListener excelListener = new ExcelListener();
         EasyExcelFactory.readBySax(fileStream, sheet, excelListener);
         return excelListener.getDatas();
      } catch (FileNotFoundException e) {
         log.error("找不到文件或文件路径错误, 文件：{}", filePath);
      }finally {
         try {
            if(fileStream != null){
               fileStream.close();
            }
         } catch (IOException e) {
            log.error("excel文件读取失败, 失败原因：{}", e);
         }
      }
      return null;
   }

   /**
    * 生成excle
    * @param filePath  绝对路径, 如：/home/chenmingjian/Downloads/aaa.xlsx
    * @param data 数据源
    * @param head 表头
    */
   public static void writeBySimple(String filePath, List<List<Object>> data, List<String> head){
      writeSimpleBySheet(filePath,data,head,null);
   }

   /**
    * 生成excle
    * @param filePath 绝对路径, 如：/home/chenmingjian/Downloads/aaa.xlsx
    * @param data 数据源
    * @param sheet excle页面样式
    * @param head 表头
    */
   public static void writeSimpleBySheet(String filePath, List<List<Object>> data, List<String> head, Sheet sheet){
      sheet = (sheet != null) ? sheet : initSheet;

      if(head != null){
         List<List<String>> list = new ArrayList<>();
         head.forEach(h -> list.add(Collections.singletonList(h)));
         sheet.setHead(list);
      }

      OutputStream outputStream = null;
      ExcelWriter writer = null;
      try {
         outputStream = new FileOutputStream(filePath);
         writer = EasyExcelFactory.getWriter(outputStream);
         writer.write1(data,sheet);
      } catch (FileNotFoundException e) {
         log.error("找不到文件或文件路径错误, 文件：{}", filePath);
      }finally {
         try {
            if(writer != null){
               writer.finish();
            }

            if(outputStream != null){
               outputStream.close();
            }

         } catch (IOException e) {
            log.error("excel文件导出失败, 失败原因：{}", e);
         }
      }

   }

   /**
    * 生成excle
    * @param filePath 绝对路径, 如：/home/chenmingjian/Downloads/aaa.xlsx
    * @param data 数据源
    */
   public static void writeWithTemplate(String filePath, List<? extends BaseRowModel> data){
      writeWithTemplateAndSheet(filePath,data,null);
   }

   /**
    * 生成excle
    * @param filePath 绝对路径, 如：/home/chenmingjian/Downloads/aaa.xlsx
    * @param data 数据源
    * @param sheet excle页面样式
    */
   public static void writeWithTemplateAndSheet(String filePath, List<? extends BaseRowModel> data, Sheet sheet){
      if(CollectionUtils.isEmpty(data)){
         return;
      }

      sheet = (sheet != null) ? sheet : initSheet;
      sheet.setClazz(data.get(0).getClass());

      OutputStream outputStream = null;
      ExcelWriter writer = null;
      try {
         outputStream = new FileOutputStream(filePath);
         writer = EasyExcelFactory.getWriter(outputStream);
         writer.write(data,sheet);
      } catch (FileNotFoundException e) {
         log.error("找不到文件或文件路径错误, 文件：{}", filePath);
      }finally {
         try {
            if(writer != null){
               writer.finish();
            }

            if(outputStream != null){
               outputStream.close();
            }
         } catch (IOException e) {
            log.error("excel文件导出失败, 失败原因：{}", e);
         }
      }

   }

   /**
    * 生成多Sheet的excle
    * @param filePath 绝对路径, 如：/home/chenmingjian/Downloads/aaa.xlsx
    * @param multipleSheelPropetys
    */
   public static void writeWithMultipleSheel(String filePath,List<MultipleSheelPropety> multipleSheelPropetys){
      if(CollectionUtils.isEmpty(multipleSheelPropetys)){
         return;
      }

      OutputStream outputStream = null;
      ExcelWriter writer = null;
      try {
         outputStream = new FileOutputStream(filePath);
         writer = EasyExcelFactory.getWriter(outputStream);
         for (MultipleSheelPropety multipleSheelPropety : multipleSheelPropetys) {
            Sheet sheet = multipleSheelPropety.getSheet() != null ? multipleSheelPropety.getSheet() : initSheet;
            if(!CollectionUtils.isEmpty(multipleSheelPropety.getData())){
               sheet.setClazz(multipleSheelPropety.getData().get(0).getClass());
            }
            writer.write(multipleSheelPropety.getData(), sheet);
         }

      } catch (FileNotFoundException e) {
         log.error("找不到文件或文件路径错误, 文件：{}", filePath);
      }finally {
         try {
            if(writer != null){
               writer.finish();
            }

            if(outputStream != null){
               outputStream.close();
            }
         } catch (IOException e) {
            log.error("excel文件导出失败, 失败原因：{}", e);
         }
      }

   }


   /*********************匿名内部类开始，可以提取出去******************************/

   @Data
   public static class MultipleSheelPropety{

      private List<? extends BaseRowModel> data;

      private Sheet sheet;
   }

   /**
    * 解析监听器，
    * 每解析一行会回调invoke()方法。
    * 整个excel解析结束会执行doAfterAllAnalysed()方法
    */
   @Getter
   @Setter
   public static class ExcelListener extends AnalysisEventListener {

      private List<Object> datas = new ArrayList<>();

      /**
       * 逐行解析
       * object : 当前行的数据
       */
      @Override
      public void invoke(Object object, AnalysisContext context) {
         //当前行
         // context.getCurrentRowNum()
         if (object != null) {
            datas.add(object);
         }
      }


      /**
       * 解析完所有数据后会调用该方法
       */
      @Override
      public void doAfterAllAnalysed(AnalysisContext context) {
         //解析结束销毁不用的资源
      }

   }

   /************************匿名内部类结束，可以提取出去***************************/

}
```

