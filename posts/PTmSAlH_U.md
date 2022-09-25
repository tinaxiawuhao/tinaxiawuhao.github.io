---
title: '上传文件到亚马逊云S3对象存储'
date: 2022-07-01 20:20:27
tags: [对象存储]
published: true
hideInList: false
feature: /post-images/PTmSAlH_U.png
isTop: false
---
### 一 在亚马逊云S3创建一个存储桶，并设置权限

#### 1 创建一个存储桶，并将权限设置为公开，因为我正常时候是用来存放网站图片

![](https://tianxiawuhao.github.io/post-images/1658665353168.png)

#### 2 配置存储卷策略

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::test.jiguiquan.com/*"
        }
    ]
}
```

![](https://tianxiawuhao.github.io/post-images/1658665363503.png)

#### 3 配置跨域策略

```json
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT",
            "POST",
            "GET"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [
            "x-amz-server-side-encryption",
            "x-amz-request-id",
            "x-amz-id-2"
        ],
        "MaxAgeSeconds": 3000
    }
]
```

![](https://tianxiawuhao.github.io/post-images/1658665372479.png)

> **到这里，存储卷的配置就算是OK啦，满足正常网站图片的使用了！**

------

### 二 编写Java类，完成文件的上传

#### 1 在项目中引入 aws-sdk 的依赖

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
    <version>1.11.347</version>
</dependency>
```

#### 2 编写上传文件的接口

```java
@Api(tags = "Auth——第三方服务模块")
@RestController
@RequiredArgsConstructor
public class ThirdpartyController {
 
    @Value("${s3.accessKeyId}")
    private String s3AccessKeyId;
    @Value("${s3.accessKeySecret}")
    private String s3AccessKeySecret;
    @Value("${s3.bucketName}")
    private String s3BucketName;
    @Value("${s3.region}")
    private String s3Region;
 
    private static BasicAWSCredentials awsCreds;
    private static AmazonS3 s3;
 
    @PostConstruct
    private void init(){
        awsCreds = new BasicAWSCredentials(s3AccessKeyId, s3AccessKeySecret);
        s3 = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(awsCreds))
                //设置服务器所属地区
                .withRegion(s3Region)
                .build();
    }
 
    @ApiResponses({@ApiResponse(code = 200, message = "文件url")})
    @ApiOperation("后端直接上传文件到亚马逊云S3")
    @PostMapping("/auth/s3/upload")
    public BaseResponse<String> uploadFileToS3(@RequestParam("file") MultipartFile file) {
        if (file.getSize() == 0){
            throw ZidanApiException.create(BmoonResponseCode.FILE_SIZE_ZERO);
        }
        if (StringUtils.isBlank(file.getOriginalFilename())){
            throw ZidanApiException.create(BmoonResponseCode.FILE_NAME_EMPTY);
        }
        // String host = "https://s3." + s3Region + ".amazonaws.com/" + s3BucketName;
        String host = "https://s3.ap-northeast-2.amazonaws.com/test.jiguiquan.com";
        String format = new SimpleDateFormat("yyyyMMdd").format(new Date());
        String uploadName = format + "/" + UUID.randomUUID().toString() + file.getOriginalFilename();
 
        ObjectMetadata metadata  = new ObjectMetadata();
        metadata.setContentType(file.getContentType());
        metadata.setContentLength(file.getSize());
 
        InputStream inputStream = null;
        try {
            inputStream = file.getInputStream();
            com.amazonaws.services.s3.model.PutObjectResult result = s3.putObject(new PutObjectRequest(s3BucketName, uploadName, inputStream, metadata));
            System.out.println(uploadName + ":文件的Md5为：" + result.getContentMd5());
            return BaseResponse.success(host + "/" + uploadName);
        }catch (Exception e) {
            e.printStackTrace();
            throw ZidanApiException.create(BmoonResponseCode.FILE_UPLOAD_FAILED);
        } finally {
            if (inputStream != null){
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

------

### 三 测试文件的上传与访问

![](https://tianxiawuhao.github.io/post-images/1658665387619.png)

浏览器访问：

![](https://tianxiawuhao.github.io/post-images/1658665394203.png)

