---
title: '工具类'
date: 2022-08-16 22:13:11
tags: [java]
published: true
hideInList: false
feature: /post-images/uZBK7g699.png
isTop: false
---
### 工具类

#### HttpUtil

```java
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import dt.logincenter.exception.BusinessException;
import dt.logincenter.exception.basic.ResponseCode;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import java.util.Iterator;
import java.util.Map;

@Component
@Slf4j
public class HttpUtil {

    @Resource
    private RestTemplate restTemplate;

    private static HttpUtil httpUtil;

    @PostConstruct
    public void init(){
        httpUtil = this;
        httpUtil.restTemplate = this.restTemplate;
    }

    /**
     * 发送请求Get请求
     *
     * @param url   请求接口
     * @param param 请求参数
     * @param paramType 请求参数类型 json：false  String：true
     */
    public JSONObject sendGet(String url, String param, boolean paramType) throws Exception {
        JSONObject jsonObject = null;
        try {
            //设置请求头
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            ResponseEntity<String> resEntity = null;
            if (!paramType) {
                HttpEntity httpEntity = new HttpEntity<>(param, headers);
                resEntity = restTemplate.exchange(url, HttpMethod.GET, httpEntity, String.class);

            }else {
                HttpEntity httpEntity = new HttpEntity<>(headers);
                resEntity = restTemplate.exchange(url, HttpMethod.GET, httpEntity, String.class, param);
            }

            jsonObject = JSONUtil.parseObj(resEntity.getBody());
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(ResponseCode.CONNECTION_TIMEOUT);
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(ResponseCode.GETDATA_FAIL);
        }
        return jsonObject;
    }

    /**
     * 发送请求Get请求
     *
     * @param url   请求接口
     * @param params 请求参数
     */
    public JSONObject sendGet(String url, Map<Object, Object> params){
        StringBuffer stringBuffer = new StringBuffer(url);

        if (params instanceof Map) {
            Iterator iterator = ((Map) params).entrySet().iterator();
            if (iterator.hasNext()) {
                stringBuffer.append("?");
                Object element;
                while (iterator.hasNext()) {
                    element = iterator.next();
                    Map.Entry<String, Object> entry = (Map.Entry) element;
                    if (entry.getValue() != null) {
                        stringBuffer.append(element).append("&");
                    }
                    url = stringBuffer.substring(0, stringBuffer.length() - 1);
                }
            }
        } else {
            throw new BusinessException(ResponseCode.REQUEST_PARAM_ERROR);
        }

        JSONObject jsonObject = null;
        try {
            //设置请求头
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            ResponseEntity<String> resEntity = null;
            HttpEntity httpEntity = new HttpEntity<>(headers);
            resEntity = restTemplate.exchange(url, HttpMethod.GET, httpEntity, String.class);

            jsonObject = JSONUtil.parseObj(resEntity.getBody());
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(ResponseCode.CONNECTION_TIMEOUT);
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(ResponseCode.GETDATA_FAIL);
        }
        return jsonObject;
    }


    /**
     * 发送请求Put请求
     *
     * @param url   请求接口
     * @param param 请求参数
     */
    public JSONObject sendPut(String url, JSONObject param) {
        JSONObject jsonObject = null;
        try {
            //设置请求头
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            HttpEntity httpEntity = new HttpEntity<>(param, headers);
            ResponseEntity<String> resEntity = restTemplate.exchange(url, HttpMethod.PUT, httpEntity, String.class);
            jsonObject = JSONUtil.parseObj(resEntity.getBody());
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(String.valueOf(ResponseCode.CONNECTION_TIMEOUT));
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(String.valueOf(ResponseCode.GETDATA_FAIL));
        }
        return jsonObject;
    }

    /**
     * 发送请求Post请求
     *
     * @param url   请求接口
     * @param param 请求参数
     */
    public JSONObject sendPost(String url, Object param) {
        JSONObject jsonObject = null;
        try {
            //设置请求头
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            HttpEntity httpEntity = new HttpEntity<>(param, headers);
            jsonObject = restTemplate.postForObject(url, httpEntity, JSONObject.class);
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(String.valueOf(ResponseCode.CONNECTION_TIMEOUT));
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(String.valueOf(ResponseCode.GETDATA_FAIL));
        }
        return jsonObject;
    }

    /**
     * 发送请求Post请求
     *
     * @param url   请求接口
     * @param param 请求参数
     */
    public JSONObject sendPost(String url, Object param,Map<String,String> header) {
        JSONObject jsonObject = null;
        try {
            //设置请求头
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            header.keySet().forEach(item->{
                headers.add(item,header.get(item));
            });

            HttpEntity httpEntity = new HttpEntity<>(param, headers);
            jsonObject = restTemplate.postForObject(url, httpEntity, JSONObject.class);
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(String.valueOf(ResponseCode.CONNECTION_TIMEOUT));
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(String.valueOf(ResponseCode.GETDATA_FAIL));
        }
        return jsonObject;
    }
    
    /**
     * 发送请求Get请求获取流文件
     *
     * @param url   请求接口
     */
    public InputStream sendGet(String url) {
        try {
            String path=ResourceUtils.getURL("classpath:").getPath();
            File file = new File(path);
            if (!file.exists())		{
                file.mkdirs();
            }
            //创造文件
            path=path+"temp.excel";
            File tempFile = new File(path);
            if (!tempFile.exists())		{
                tempFile.createNewFile();
            }
            ResponseEntity<byte[]> rsp = restTemplate.getForEntity(url, byte[].class);
            System.out.println("文件下载请求结果状态码：" + rsp.getStatusCode());
            // 将下载下来的文件内容保存到本地
            Files.write(Paths.get(path.substring(1)), Objects.requireNonNull(rsp.getBody(), "未获取到下载文件"));
            return new FileInputStream(path);
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(ResponseCode.CONNECTION_TIMEOUT);
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(ResponseCode.GETDATA_FAIL);
        }

    }

      /**
     * 发送请求Get请求获取流文件
     *
     * @param url   请求接口
     */
    public InputStream sendGet(String url) {
        try {
            ResponseEntity<byte[]> entity = restTemplate.getForEntity(url, byte[].class);
            Objects.requireNonNull(entity.getBody());
            String path=ResourceUtils.getURL("classpath:").getPath()+"temp.excel";
            BufferedOutputStream bos;
            FileOutputStream fos;
            File file;
            File dir = new File(path);
            if (!dir.exists()) {// 判断文件目录是否存在
                dir.mkdirs();
            }
            file = new File(path);
            fos = new FileOutputStream(file);
            bos = new BufferedOutputStream(fos);
            bos.write(Objects.requireNonNull(entity.getBody()));
            bos.close();
            fos.close();

            return new FileInputStream(file);
        } catch (Exception e) {
            if (e.toString().contains("timed out")) {
                throw new BusinessException(ResponseCode.CONNECTION_TIMEOUT);
            }
            log.error("接口调用失败{}",e);
            throw new BusinessException(ResponseCode.GETDATA_FAIL);
        }

    }
}
```

#### JavaScriptUtil

```java
import org.apache.tomcat.util.buf.StringUtils;

import javax.script.*;
import java.util.Map;
import java.util.Set;

/**
 * java执行javaScript代码的工具类
 *
 * @author wuhao
 */
public class JavaScriptUtil {

    /**
     * 单例的JavaScript解析引擎
     */
    private static final ScriptEngine javaScriptEngine;

    static {
        ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine scriptEngine = manager.getEngineByName("js");
        if (scriptEngine == null) {
            throw new RuntimeException("获取JavaScript解析引擎失败");
        }
        javaScriptEngine = scriptEngine;
    }


    /**
     * 执行一段JavaScript代码
     *
     * @param script JavaScript的代码
     * @return JavaScript代码运行结果的值
     * @throws ScriptException JavaScript代码运行异常
     */
    public static Object execute(String script) throws ScriptException {
        return javaScriptEngine.eval(script);
    }


    /**
     * 运行一个JavaScript代码段,并获取指定变量名的值
     *
     * @param script        代码段
     * @param attributeName 已知的变量名
     * @return 指定变量名对应的值
     * @throws ScriptException JavaScript代码运行异常
     */
    public static Object executeForAttribute(String script, String attributeName) throws ScriptException {
        javaScriptEngine.eval(script);
        return javaScriptEngine.getContext().getAttribute(attributeName);
    }

    /**
     * 获取当前语句运行后第一个有值变量的值
     *
     * @param script 代码段
     * @return 第一个有值变量的值
     * @throws ScriptException JavaScript代码运行异常
     */
    public static Object executeForFirstAttribute(String script) throws ScriptException {

        //这里重新获取一个JavaScript解析引擎是为了避免代码中有其他调用工具类的地方的变量干扰
        //重新获取后,这个JavaScript解析引擎只执行了这次传入的代码,不会保存其他地方的变量
        //全局的解析器中会保存最大200个变量,JavaScript解析引擎本身最大保存100个变量
        ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine scriptEngine = manager.getEngineByName("js");
        if (scriptEngine == null) {
            throw new RuntimeException("获取JavaScript解析引擎失败");
        }

        scriptEngine.eval(script);
        ScriptContext context = scriptEngine.getContext();
        if (context == null) {
            return null;
        }
        Bindings bindings = context.getBindings(ScriptContext.ENGINE_SCOPE);
        if (bindings == null) {
            return null;
        }
        Set<Map.Entry<String, Object>> entrySet = bindings.entrySet();
        if (entrySet.isEmpty()) {
            return null;
        }
        for (Map.Entry<String, Object> entry : entrySet) {
            if (entry.getValue() != null) {
                return entry.getValue();
            }
        }
        return null;
    }

    /**
     * 执行一段JavaScript代码，传入参数，获取返回值
     * String script = "Math.pow(x, y)";
     * Map<String, Object> params = Maps.newHashMap();
     * params.put("x", 10);
     * params.put("y", 2);
     * <p>
     * String script = ”if (company == 1) {
     * return true;
     * } else if (company == 2 && amt > 100) {
     * return true;
     * } else if (company == 3 && amt > 200) {
     * return true;
     * } else {
     * return false;
     * }“
     * <p>
     * params.put("company", 1);
     * params.put("amt", 100);
     *
     * @param script
     * @param params
     * @return
     */
    public static String execute(String script, Map<String, Object> params) {
        try {
            String paramsName = StringUtils.join(params.keySet(), ',');
            String executeScript = String.format("function execute(%s) { %s }", paramsName, script);
            javaScriptEngine.eval(executeScript);
            Invocable jsInvoke = (Invocable) javaScriptEngine;
            Object execute = jsInvoke.invokeFunction("execute", params.values().toArray());
            return execute.toString();
        } catch (ScriptException | NoSuchMethodException e) {
            System.out.println("表达式runtime错误:" + e.getMessage());
        }
        return null;
    }
}
```

#### MinioUtil

```java

import io.minio.*;
import io.minio.errors.*;
import io.minio.http.Method;
import io.minio.messages.Item;
import io.minio.messages.Upload;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;

import javax.crypto.SecretKey;
import java.io.IOException;
import java.io.InputStream;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.ArrayList;
import java.util.List;

@Component
@Slf4j
public class MinioUtil {
    @Autowired
    private MinioClient minioClient;

    private static final int DEFAULT_EXPIRY_TIME = 7 * 24 * 3600;

    /**
     * 检查存储桶是否存在
     *
     * @param bucketName 存储桶名称
     * @return
     * @throws Exception
     */
    public boolean bucketExists(String bucketName)  {
        try {
            return minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());
        } catch (Exception e) {
            log.error("mino 服务端连接异常",e);
            throw new RuntimeException("mino 服务端连接异常");
        }
    }

    /**
     * 创建存储桶
     *
     * @param bucketName 存储桶名称
     * @return
     * @throws Exception
     */
    public boolean makeBucket(String bucketName) throws Exception {
        boolean flag = bucketExists(bucketName);
        if (!flag) {
            minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucketName).build());
            return true;
        }
        return false;
    }


    /**
     * 文件上传
     *
     * @param bucketName
     * @param multipartFile
     */
    public void putObject(String bucketName, MultipartFile multipartFile, String filename, SecretKey secretKey) throws Exception {
        makeBucket(bucketName);
        PutObjectOptions putObjectOptions = new PutObjectOptions(multipartFile.getSize(), PutObjectOptions.MIN_MULTIPART_SIZE);
        putObjectOptions.setContentType(multipartFile.getContentType());
        if (null != secretKey) {
            putObjectOptions.setSse(ServerSideEncryption.withCustomerKey(secretKey));
        }
        minioClient.putObject(bucketName, filename, multipartFile.getInputStream(), putObjectOptions);
    }


    /**
     * 普通文件上传
     * @param bucketName
     * @param \
     */
    public void putObject(String bucketName, InputStream inputStream, String filename, SecretKey secretKey,long fileSize) throws Exception {
        PutObjectOptions putObjectOptions = new PutObjectOptions(fileSize, 0);
        minioClient.putObject(bucketName, filename, inputStream, putObjectOptions);
    }

    /**
     * 分享。
     *
     * @param bucketName 存储桶名称
     * @param objectName 存储桶里的对象名称
     * @param expires    失效时间（以秒为单位），默认是7天，不得大于七天
     * @return
     */
    public String getPresignedObjectUrl(String bucketName, String objectName, Integer expires) throws Exception {
        boolean flag = bucketExists(bucketName);
        if (flag) {
            if (null == expires) {
                expires = DEFAULT_EXPIRY_TIME;
            }
            if (expires < 1 || expires > DEFAULT_EXPIRY_TIME) {
                throw new RuntimeException("expires must be in range of 1 to " + DEFAULT_EXPIRY_TIME);
            }
            try {
                return minioClient.getPresignedObjectUrl(Method.GET, bucketName, objectName, expires, null);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return null;
    }

    /**
     * 获取一个临时可以上传的url
     * @param bucketName
     * @param objectName
     * @return
     * @throws Exception
     */
    public String presignedPutObject(String bucketName, String objectName) {
        try {
            return minioClient.presignedPutObject(bucketName, objectName);
        } catch (Exception e) {
            throw new RuntimeException("获取临时文件上传地址异常");
        }
    }

    /**
     * 列出某个存储桶中的所有对象
     *
     * @param bucketName
     * @param objectNamePrefix 前缀
     * @return
     * @throws Exception
     */
    public List<ComposeSource> listObjectsSource(String bucketName, String objectNamePrefix) throws Exception {
        List<ComposeSource> sources = new ArrayList<>();
        Iterable<Result<Item>> results = minioClient.listObjects(bucketName, objectNamePrefix);
        for (Result<Item> result : results) {
            Item item = result.get();
            sources.add(ComposeSource.builder().bucket(bucketName).object(objectNamePrefix).build());


        }

//        minioClient.listIncompleteUploads()
        return sources;
    }

    /**
     * 文件合并
     *
     * @param bucketName
     * @param objectName
     * @param sources
     * @param secretKey
     * @throws Exception
     */
    public void composeObject(String bucketName, String objectName, List<ComposeSource> sources, SecretKey secretKey) throws Exception {
        List<ComposeSource> sourceObjectList = new ArrayList<>(sources.size());
        for (int i=0;i<sources.size();i++){
           sourceObjectList.add(
                    ComposeSource.builder()
                            .bucket(bucketName)
                            .object(sources.get(i).object()+"_" + i)
                            .build()
            );
        }
        ObjectWriteResponse objectWriteResponse = minioClient.composeObject(
                ComposeObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .sources(sourceObjectList)
                        .build());
        for (ComposeSource source : sourceObjectList) {
            minioClient.removeObject(source.bucket(), source.object());
        }
    }

    /**
     * 查询已经上传成功的文件
     * @param bucketName
     * @param prefix
     */
    public List<Integer>  listIncompleteUploads(String bucketName,String prefix){
        List<Integer> rslt=new ArrayList<>();
        try {
            Iterable<Result<Item>> results=minioClient.listObjects(
                    ListObjectsArgs.builder()
                            .bucket(bucketName)
                            .prefix(prefix)
                            .maxKeys(1000)
                            .build());
            for (Result<Item> result : results) {
                String obectName = result.get().objectName();
                String[] split = StringUtils.split(obectName, "_");
                rslt.add(Integer.parseInt(split[split.length-1]));
            }
        }catch (Exception e) {
            log.error("调用minio查询已经分片上传信息异常",e);
            throw new RuntimeException("调用minio查询已经分片上传信息异常");
        }
        return rslt;
    }
}
```

