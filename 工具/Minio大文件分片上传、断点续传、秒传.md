[TOC]

# Minio大文件分片上传、断点续传、秒传

## 前后端配合完成方式

### 文件上传流程图

[流程图](https://www.processon.com/view/link/63fc58c9621e3702bf11d552)

### 参考资料

[minio-upload](https://gitee.com/Gary2016/minio-upload)

- 拉取代码跑起来直接可以验证的项目

[Minio进阶 - Minio+vue-uploader 分片上传方案及案例详解](https://java.isture.com/arch/minio/minio-uploader.html)

- 这里面写了整体思路、流程图

### Docker创建Minio

```shell
docker run -d \
  -p 9000:9000 \
  -p 9001:9001 \
  --name minio \
  -v ~/Documents/Docker/minio:/data \
  -e "MINIO_ROOT_USER=agefades" \
  -e "MINIO_ROOT_PASSWORD=agefades" \
  minio/minio server /data --console-address ":9001"
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1676274595458.png)

- 默认访问地址: ip + port（localhost:9001）
- 账号密码: docker启动命令中设置的环境变量

#### 创建AccessKey&SecretKey

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1676275488883.png)

#### 创建bucket

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1676528498180.png)

### SpringBoot整合Minio

#### pom.xml

```xml
<!-- minio -->
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-java-sdk-s3</artifactId>
  <!-- 这里最好不要手动指定版本，SpringBoot有配套指定版本 -->
  <!-- <version>1.12.404</version> -->
</dependency>
```

#### application.yml

```yaml
minio:
  endpoint: http://127.0.0.1:9000
  accessKey: s2vVtzvQXqx3Cvut
  secretKey: 6mdJITd331XXctFT0nUQMQmQrgcIMtcD
  bucketName: agefades
```

#### 配置类

```java
package com.agefades.log.common.file.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

/**
 * minio配置类
 */
@Data
@Configuration
@ConfigurationProperties(prefix = "minio")
public class MinioProperties {

    /**
     * OSS 访问端点，集群时需提供统一入口
     */
    String endpoint;

    /**
     * 用户名
     */
    String accessKey;

    /**
     * 密码
     */
    String secretKey;

    /**
     * 存储桶名
     */
    String bucketName;
}
```

```java
package com.agefades.log.common.file.config;

import com.amazonaws.ClientConfiguration;
import com.amazonaws.Protocol;
import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.client.builder.AwsClientBuilder;
import com.amazonaws.regions.Regions;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * oss客户端
 *
 * @author DuChao
 * @since 2023/2/15 15:04
 */
@Configuration
public class AmazonS3Config {

    @Autowired
    private MinioProperties minioProperties;

    @Bean(name = "amazonS3Client")
    public AmazonS3 amazonS3Client () {
        // 设置连接时的参数
        ClientConfiguration config = new ClientConfiguration();
        // 设置连接方式为HTTP，可选参数为HTTP和HTTPS
        config.setProtocol(Protocol.HTTP);
        // 设置网络访问超时时间
        config.setConnectionTimeout(5000);
        config.setUseExpectContinue(true);
        AWSCredentials credentials = new BasicAWSCredentials(minioProperties.getAccessKey(), minioProperties.getSecretKey());
        // 设置Endpoint
        AwsClientBuilder.EndpointConfiguration end_point = new AwsClientBuilder.EndpointConfiguration(minioProperties.getEndpoint(), Regions.US_EAST_1.name());
        return AmazonS3ClientBuilder.standard()
                .withClientConfiguration(config)
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withEndpointConfiguration(end_point)
                .withPathStyleAccessEnabled(true).build();
    }

}
```

#### 表SQL

```sql
DROP TABLE IF EXISTS `oss_file`;
CREATE TABLE `oss_file`
(
    `id`              bigint       NOT NULL,
    `upload_id`       varchar(255) NOT NULL COMMENT '分片上传的uploadId',
    `file_identifier` varchar(500) NOT NULL COMMENT '文件唯一标识（md5）',
    `file_name`       varchar(500) NOT NULL COMMENT '文件名',
    `object_key`      varchar(500) NOT NULL COMMENT '文件的key',
    `total_size`      bigint       NOT NULL COMMENT '文件大小（byte）',
    `chunk_size`      bigint       NOT NULL COMMENT '每个分片大小（byte）',
    `chunk_num`       int          NOT NULL COMMENT '分片数量',
    `is_finish`       int          NOT NULL COMMENT '是否已完成上传(完成合并),1是0否',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uq_file_identifier` (`file_identifier`) USING BTREE,
    UNIQUE KEY `uq_upload_id` (`upload_id`) USING BTREE
) ENGINE=InnoDB COMMENT='上传文件记录表';
```

#### 实体类

```java
package com.agefades.log.common.file.entity;

import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Builder;
import lombok.Data;

/**
 * 上传文件记录表
 *
 * @author DuChao
 * @since 2023/2/15 15:03
 */
@Data
@Builder
@TableName("oss_file")
public class OssFile {

    /**
     * 主键
     */
    @TableId
    private String id;
    /**
     * 分片上传的uploadId
     */
    private String uploadId;
    /**
     * 文件唯一标识（md5）
     */
    private String fileIdentifier;
    /**
     * 文件名
     */
    private String fileName;
    /**
     * 文件的key
     */
    private String objectKey;
    /**
     * 文件大小（byte）
     */
    private Long totalSize;
    /**
     * 每个分片大小（byte）
     */
    private Long chunkSize;
    /**
     * 分片数量
     */
    private Integer chunkNum;
    /**
     * 是否已完成上传(完成合并),1是0否
     */
    private Integer isFinish;

}
```

#### 数据传输对象

```java
package com.agefades.log.common.file.resp;

import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;
import lombok.Builder;
import lombok.Data;

import java.util.Map;

/**
 * 文件上传进度响应对象
 *
 * @author DuChao
 * @since 2023/2/15 15:19
 */
@Data
@Builder
@ApiModel("文件上传进度对象")
public class UploadProgress {

    @ApiModelProperty("是否为新文件（从未上传过的文件）,1是0否")
    private Integer isNew;

    @ApiModelProperty("是否已完成上传（是否已经合并分片）,1是0否")
    private Integer isFinish;

    @ApiModelProperty("文件地址")
    private String path;

    @ApiModelProperty("未完全上传时,还未上传的(分片、上传链接)Map")
    private Map<String, String> undoneChunkMap;

}
```

```java
package com.agefades.log.common.file.req;

import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;

/**
 * 创建分片上传请求对象
 *
 * @author DuChao
 * @since 2023/2/16 10:39
 */
@Data
@ApiModel("创建分片上传请求对象")
public class CreateMultipartUpload {

    @ApiModelProperty("文件唯一标识(MD5)")
    @NotBlank(message = "文件标识不能为空")
    private String identifier;

    @ApiModelProperty("文件名称")
    @NotBlank(message = "文件名称不能为空")
    private String fileName;

    @ApiModelProperty("文件大小（byte）")
    @NotNull(message = "文件大小不能为空")
    private Long totalSize;

    @ApiModelProperty("分片大小（byte）")
    @NotNull(message = "分片大小不能为空")
    private Long chunkSize;

}
```

#### Mapper

```java
package com.agefades.log.common.file.mapper;

import com.agefades.log.common.file.entity.OssFile;
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface OssFileMapper extends BaseMapper<OssFile> {



}
```

#### Service

```java
package com.agefades.log.common.file.service;

import com.agefades.log.common.file.entity.OssFile;
import com.agefades.log.common.file.req.CreateMultipartUpload;
import com.agefades.log.common.file.resp.UploadProgress;
import com.baomidou.mybatisplus.extension.service.IService;

import java.util.Map;

public interface OssFileService extends IService<OssFile> {

    /**
     * 获取文件上传进度
     */
    UploadProgress getUploadProgress(String identifier);

    /**
     * 发起分片上传
     */
    Map<String, String> createMultipartUpload(CreateMultipartUpload req);

    /**
     * 合并分片文件
     */
    String merge(String identifier);
}
```

#### SerivceImpl

```java
package com.agefades.log.common.file.service.impl;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.DateTime;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import com.agefades.log.common.core.enums.BoolEnum;
import com.agefades.log.common.core.util.Assert;
import com.agefades.log.common.file.config.MinioProperties;
import com.agefades.log.common.file.entity.OssFile;
import com.agefades.log.common.file.mapper.OssFileMapper;
import com.agefades.log.common.file.req.CreateMultipartUpload;
import com.agefades.log.common.file.resp.UploadProgress;
import com.agefades.log.common.file.service.OssFileService;
import com.amazonaws.HttpMethod;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.model.CompleteMultipartUploadRequest;
import com.amazonaws.services.s3.model.GeneratePresignedUrlRequest;
import com.amazonaws.services.s3.model.InitiateMultipartUploadRequest;
import com.amazonaws.services.s3.model.InitiateMultipartUploadResult;
import com.amazonaws.services.s3.model.ListPartsRequest;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PartETag;
import com.amazonaws.services.s3.model.PartListing;
import com.amazonaws.services.s3.model.PartSummary;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.http.MediaTypeFactory;
import org.springframework.stereotype.Service;

import java.net.URL;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.stream.Collectors;

@Slf4j
@Service
public class OssFileServiceImpl extends ServiceImpl<OssFileMapper, OssFile> implements OssFileService {

    /**
     * 预签名url过期时间: 10分钟
     */
    private static final Integer PRE_SIGN_URL_EXPIRE = 10;

    @Autowired
    MinioProperties minioProperties;

    @Autowired
    private AmazonS3 amazonS3;

    @Override
    public UploadProgress getUploadProgress(String identifier) {
        // 准备响应对象
        UploadProgress.UploadProgressBuilder builder = UploadProgress.builder();

        OssFile ossFile = getOssFileByIdentifier(identifier);
        // 文件在库中不存在,即为新文件
        if (ObjectUtil.isNull(ossFile)) {
            builder.isNew(BoolEnum.Y.getCode());
            return builder.build();
        }

        Integer isFinish = ossFile.getIsFinish();
        builder.isFinish(isFinish);
        // 判断是否已经完成上传
        if (ObjectUtil.equal(isFinish, BoolEnum.Y.getCode())) {
            // 已上传,拼接路径直接返回
            builder.path(getPath(ossFile.getObjectKey()));
            return builder.build();
        }

        // 未完成上传(断点续传情形),利用oss客户端获取已上传的分片
        ListPartsRequest listPartsRequest = new ListPartsRequest(minioProperties.getBucketName(), ossFile.getObjectKey(), ossFile.getUploadId());
        PartListing partListing = amazonS3.listParts(listPartsRequest);

        if (ObjectUtil.isNotNull(partListing)) {
            // 响应的分块Map对象
            TreeMap<String, String> chunkMap = new TreeMap<>();

            // 计算所有分片
            List<Integer> allPartNumbers = new ArrayList<>();
            Integer chunkNum = ossFile.getChunkNum();
            for (int i = 1; i <= chunkNum; i++) {
                allPartNumbers.add(i);
            }

            List<PartSummary> parts = partListing.getParts();
            // 文件服务器中没有分片数据时(可能刚创建分片上传请求,还没进行上传)
            if (CollUtil.isEmpty(parts)) {
                allPartNumbers.forEach(v -> {
                    // 未上传的，value为预上传链接
                    String uploadUrl = genPreSignUploadUrl(ossFile.getObjectKey(), ossFile.getUploadId(), v);
                    chunkMap.put("chunk_" + (v - 1), uploadUrl);
                });
            } else {
                // 已上传的分片
                List<Integer> finishPartNumbers = parts.stream().map(PartSummary::getPartNumber).collect(Collectors.toList());

                // 遍历所有分片 获取所有未上传分片的预上传链接
                allPartNumbers.forEach(v -> {
                    // 已上传的，value为finish，标记为，用于前端判断跳过
                    if (CollUtil.contains(finishPartNumbers, v)) {
                        chunkMap.put("chunk_" + (v - 1), "finish");
                    } else {
                        // 未上传的，value为预上传链接
                        String uploadUrl = genPreSignUploadUrl(ossFile.getObjectKey(), ossFile.getUploadId(), v);
                        chunkMap.put("chunk_" + (v - 1), uploadUrl);
                    }
                });
            }
            builder.chunkMap(chunkMap);
        }
        return builder.build();
    }


    @Override
    public Map<String, String> createMultipartUpload(CreateMultipartUpload req) {
        // 校验
        String identifier = req.getIdentifier();
        OssFile ossFile = getOssFileByIdentifier(identifier);
        Assert.isNull(ossFile, "该文件已发起过分片上传任务");

        // 构造存库文件对象
        Long totalSize = req.getTotalSize();
        Long chunkSize = req.getChunkSize();
        String fileName = req.getFileName();
        OssFile.OssFileBuilder builder = OssFile.builder()
                .fileIdentifier(identifier)
                .fileName(fileName)
                .totalSize(totalSize)
                .chunkSize(chunkSize)
                .isFinish(BoolEnum.N.getCode());

        // 计算分片数
        int chunkNum = (int) Math.ceil(totalSize * 1.0 / chunkSize);
        builder.chunkNum(chunkNum);

        // 拼接文件key
        String suffix = StrUtil.subAfter(fileName, ".", true);
        String objectKey = LocalDate.now() + "/" + IdUtil.fastSimpleUUID() + "." + suffix;
        builder.objectKey(objectKey);

        // oss客户端发起分片上传任务,获取uploadId
        String contentType = MediaTypeFactory.getMediaType(objectKey).orElse(MediaType.APPLICATION_OCTET_STREAM).toString();
        ObjectMetadata objectMetadata = new ObjectMetadata();
        objectMetadata.setContentType(contentType);
        InitiateMultipartUploadResult result = amazonS3
                .initiateMultipartUpload(new InitiateMultipartUploadRequest(minioProperties.getBucketName(), objectKey)
                        .withObjectMetadata(objectMetadata));
        String uploadId = result.getUploadId();
        builder.uploadId(uploadId);

        // 存库
        save(builder.build());

        // 获取每个分片的预签名上传地址
        TreeMap<String, String> map = new TreeMap<>();
        for (int i = 1; i <= chunkNum; i++) {
            String uploadUrl = genPreSignUploadUrl(objectKey, uploadId, i);
            map.put("chunk_" + (i - 1), uploadUrl);
        }
        return map;
    }

    @Override
    public String merge(String identifier) {
        // 校验
        OssFile ossFile = getOssFileByIdentifier(identifier);
        Assert.notNull(ossFile, "该文件未发起过分片上传任务");
        Assert.isFalse(ObjectUtil.equal(BoolEnum.Y.getCode(), ossFile.getIsFinish()), "该文件已完成合并");

        ListPartsRequest listPartsRequest = new ListPartsRequest(minioProperties.getBucketName(), ossFile.getObjectKey(), ossFile.getUploadId());
        PartListing partListing = amazonS3.listParts(listPartsRequest);
        List<PartSummary> parts = partListing.getParts();
        Assert.isTrue(ObjectUtil.equal(ossFile.getChunkNum(), CollUtil.size(parts)), "分片缺失,请重新上传");

        // 发起合并请求
        CompleteMultipartUploadRequest completeMultipartUploadRequest = new CompleteMultipartUploadRequest()
                .withUploadId(ossFile.getUploadId())
                .withKey(ossFile.getObjectKey())
                .withBucketName(minioProperties.getBucketName())
                .withPartETags(parts.stream().map(partSummary -> new PartETag(partSummary.getPartNumber(), partSummary.getETag())).collect(Collectors.toList()));
        amazonS3.completeMultipartUpload(completeMultipartUploadRequest);

        // 更新数据状态
        ossFile.setIsFinish(BoolEnum.Y.getCode());
        updateById(ossFile);
        return getPath(ossFile.getObjectKey());
    }

    /**
     * 获取分片预签名上传地址
     */
    private String genPreSignUploadUrl(String objectKey, String uploadId, Integer partNumber) {
        // 计算到期时间
        DateTime expireDate = DateUtil.offsetMinute(new Date(), PRE_SIGN_URL_EXPIRE);
        // 构建请求对象
        GeneratePresignedUrlRequest request = new GeneratePresignedUrlRequest(minioProperties.getBucketName(), objectKey)
                .withExpiration(expireDate).withMethod(HttpMethod.PUT);
        // 补充分片相关参数
        request.addRequestParameter("uploadId", uploadId);
        request.addRequestParameter("partNumber", partNumber.toString());
        // 获取链接
        URL preSignedUrl = amazonS3.generatePresignedUrl(request);
        return preSignedUrl.toString();
    }

    /**
     * 通过文件唯一标识(MD5)获取文件数据
     */
    private OssFile getOssFileByIdentifier(String identifier) {
        return lambdaQuery()
                .eq(OssFile::getFileIdentifier, identifier)
                .one();
    }

    /**
     * 获取文件路径
     */
    private String getPath(String objectKey) {
        return StrUtil.format("{}/{}/{}", minioProperties.getEndpoint(), minioProperties.getBucketName(), objectKey);
    }

}
```

#### Controller

```java
package com.agefades.log.system.service.controller;

import com.agefades.log.common.core.base.Result;
import com.agefades.log.common.file.req.CreateMultipartUpload;
import com.agefades.log.common.file.resp.UploadProgress;
import com.agefades.log.common.file.service.OssFileService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import io.swagger.annotations.ApiParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;
import java.util.Map;

@Api(tags = "文件API")
@RestController
@RequestMapping("/file")
public class OssFileController {

    @Autowired
    private OssFileService ossFileService;

    @ApiOperation(value = "获取上传进度",notes = "根据响应参数标识字段[isNew]/[isFinish]判断: " +
            "1.新文件时,需调用[发起分片上传],2.已上传完成文件,返回文件地址," +
            "3.未完全上传文件,返回分片Map对象,key=chunk_{分片下标},value=finish表示当前该分片已上传,value=预上传url链接，表示当前该分片未上传")
    @GetMapping("/{identifier}")
    public Result<UploadProgress> getUploadProgress(@ApiParam("文件MD5值") @PathVariable("identifier") String identifier) {
        return Result.success(ossFileService.getUploadProgress(identifier));
    }

    @ApiOperation(value = "发起分片上传", notes = "返回文件每个分片及对应的预上传链接Map")
    @PostMapping("/createMultipartUpload")
    public Result<Map<String, String>> createMultipartUpload(@Valid @RequestBody CreateMultipartUpload req) {
       return Result.success(ossFileService.createMultipartUpload(req));
    }

    @ApiOperation(value = "合并分片文件", notes = "文件各分片上传完成后,调用此接口进行合并,成功则返回文件公网可访问地址")
    @PostMapping("/merge/{identifier}")
    public Result<String> merge(@ApiParam("文件MD5值") @PathVariable("identifier") String identifier) {
        return Result.success(ossFileService.merge(identifier));
    }

}
```

### 测试效果

- 使用此前后端项目进行的测试 [minio-upload](https://gitee.com/Gary2016/minio-upload)
- 该项目里有演示动图，直接看该项目的即可，此处写该标题是本人实际使用验证过
- 上述后端相关代码是本人借鉴各处资料，用符合自己项目的写法写过，明白整体方案思路即可

## AWS STS临时凭证方式

- 这种方式网上搜资料很少，很容易有各种异常问题，耗费时间精力比较多，且风险大
- 此种方式，经测试，前端 minioClient.putObject() 方法调用，对大文件会进行自动切片上传并最后发起合并结束
  - 但切片与上传的流程是串行，切一片上传一片、再切一片再上传一片
  - 讨论过多种方式（改源码、手动切多线程调...），最终觉得风险和不可控因素较高
- 此种方式，经测试，不会自动支持 极速秒传（已经上传过的文件内容完全相同的文件可以直接返回路径）
  - 也不会支持 断点续传（1G文件上传到99%时，浏览器崩溃，下次再传会从0开始）

### 参考资料

[minio官方文档中对STS的介绍](https://min.io/docs/minio/linux/developers/security-token-service/AssumeRoleWithWebIdentity.html)

[minio官方文档中对权限策略支持的配置项](https://min.io/docs/minio/linux/administration/identity-access-management/policy-based-access-control.html#minio-policy)

[参考的博客链接](https://blog.csdn.net/yilvqingtai/article/details/126059858)

### AWS S3客户端的方式获取临时凭证

#### pom.xml

- 版本都是SpringBoot管理的，不要特殊指定版本

```xml
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-java-sdk-s3</artifactId>
</dependency>

<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-java-sdk-sts</artifactId>
</dependency>
```

#### AwsStsUtil

- 这里面的权限指定适用于 普通文件读写、大文件上传的分片、合并等操作

```java
package com.agefades.log.common.file.util;

import com.agefades.log.common.core.util.Assert;
import com.agefades.log.common.core.util.SpringUtil;
import com.agefades.log.common.file.config.MinioProperties;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.client.builder.AwsClientBuilder;
import com.amazonaws.services.securitytoken.AWSSecurityTokenService;
import com.amazonaws.services.securitytoken.AWSSecurityTokenServiceClientBuilder;
import com.amazonaws.services.securitytoken.model.AssumeRoleRequest;
import com.amazonaws.services.securitytoken.model.AssumeRoleResult;
import com.amazonaws.services.securitytoken.model.Credentials;
import lombok.extern.slf4j.Slf4j;

/**
 * Aws STS 临时凭证获取工具类
 *
 * @author DuChao
 * @since 2023/2/28 10:52
 */
@Slf4j
public class AwsStsUtil {

    private static final MinioProperties properties = SpringUtil.getBean(MinioProperties.class);

    public static final String ROLE_ARN = "arn:aws:s3:::" + properties.getBucketName() + "/*";
    public static final String ROLE_SESSION_NAME = "anysession";
    public static final String POLICY = "{\r\n"
            + "    \"Version\": \"2012-10-17\",\r\n"
            + "    \"Statement\": [\r\n"
            + "        {\r\n"
            + "            \"Effect\": \"Allow\",\r\n"
            + "            \"Action\": [\r\n"
            + "                \"s3:PutObject\",\r\n"
            + "                \"s3:GetBucketLocation\",\r\n"
            + "                \"s3:AbortMultipartUpload\",\r\n"
            + "                \"s3:ListMultipartUploadParts\",\r\n"
            + "                \"s3:ListBucketMultipartUploads\",\r\n"
            + "                \"s3:ListBucket\"\r\n"
//    		+ "                \"s3:*\"\r\n"
            + "            ],\r\n"
            + "            \"Resource\": [\r\n"
            + "                \"arn:aws:s3:::" + properties.getBucketName() + "/*\"\r\n"
            + "            ]\r\n"
            + "        }\r\n"
            + "    ]\r\n"
            + "}";

    public static Credentials createSts() {
        BasicAWSCredentials awsCredentials = new BasicAWSCredentials(properties.getAccessKey(), properties.getSecretKey());
        AwsClientBuilder.EndpointConfiguration regionEndpointConfig = new AwsClientBuilder.EndpointConfiguration(properties.getEndpoint(), null);
        AWSSecurityTokenService stsClient = AWSSecurityTokenServiceClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(awsCredentials))
                .withEndpointConfiguration(regionEndpointConfig)
                .build();

        AssumeRoleRequest assumeRoleRequest = new AssumeRoleRequest();
        assumeRoleRequest.setRoleArn(ROLE_ARN);
        assumeRoleRequest.setPolicy(POLICY);
        assumeRoleRequest.setRoleSessionName(ROLE_SESSION_NAME);
        assumeRoleRequest.setDurationSeconds(3600); //3600

        AssumeRoleResult assumeRoleResult = stsClient.assumeRole(assumeRoleRequest);
        Assert.notNull(assumeRoleResult, "获取临时凭证失败,请开发人员排查");
        return assumeRoleResult.getCredentials();
    }

}
```

#### 返回示例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1677580627231.png)

### Minio客户端的方式获取临时凭证

#### pom.xml

- 我这里使用的版本

```xml
<minio.version>8.3.0</minio.version>
<okhttp3.version>4.8.1</okhttp3.version>
```

```xml
<!-- minio -->
<dependency>
  <groupId>io.minio</groupId>
  <artifactId>minio</artifactId>
  <version>${minio.version}</version>
</dependency>

<dependency>
  <groupId>com.squareup.okhttp3</groupId>
  <artifactId>okhttp</artifactId>
  <version>${okhttp3.version}</version>
</dependency>
```

#### 测试demo

```java
package com.agefades.log.system.service.controller;

import io.minio.MinioClient;
import io.minio.PutObjectArgs;
import io.minio.credentials.AssumeRoleProvider;
import io.minio.credentials.Credentials;
import io.minio.credentials.Provider;
import io.minio.credentials.StaticProvider;
import io.minio.messages.Bucket;

import java.io.File;
import java.io.FileInputStream;
import java.util.List;
import java.util.stream.Collectors;

public class Demo {

    public static final String ENDPOINT = "http://127.0.0.1:9000/";
    public static final String ACCESS_KEY_COMPANY = "agefades";
    public static final String SECRET_KEY_COMPANY = "agefades";
    public static final String REGION = "cn-north-1";
    public static final String BUCKET = "agefades";
    public static final String ROLE_ARN = "arn:aws:s3:::agefades/*";
    public static final String ROLE_SESSION_NAME = "anysession";
    public static final String POLICY_ALL = "{\r\n"
            + "    \"Version\": \"2012-10-17\",\r\n"
            + "    \"Statement\": [\r\n"
            + "        {\r\n"
            + "            \"Effect\": \"Allow\",\r\n"
            + "            \"Action\": [\r\n"
            + "                \"s3:PutObject\",\r\n"
            + "                \"s3:GetBucketLocation\",\r\n"
            + "                \"s3:AbortMultipartUpload\",\r\n"
            + "                \"s3:ListMultipartUploadParts\",\r\n"
            + "                \"s3:ListBucketMultipartUploads\",\r\n"
            + "                \"s3:ListBucket\"\r\n"
//    		+ "                \"s3:*\"\r\n"
            + "            ],\r\n"
            + "            \"Resource\": [\r\n"
            + "                \"arn:aws:s3:::agefades/*\"\r\n"
            + "            ]\r\n"
            + "        }\r\n"
            + "    ]\r\n"
            + "}";

    public static void main(String[] args) throws Exception {

        // create AssumeRoleProvider instance.
        Provider provider = new AssumeRoleProvider(
                ENDPOINT,
                ACCESS_KEY_COMPANY,
                SECRET_KEY_COMPANY,
                36000,
                POLICY_ALL,
                REGION,
                ROLE_ARN,
                ROLE_SESSION_NAME,
                null,
                null);

        Credentials credentials = provider.fetch();
        System.out.println(credentials.accessKey());
        System.out.println(credentials.secretKey());
        System.out.println(credentials.sessionToken());
        System.out.println(credentials.isExpired());

        StaticProvider staticProvider = new StaticProvider(credentials.accessKey(), credentials.secretKey(), credentials.sessionToken());
        MinioClient minioClient = MinioClient.builder().endpoint(ENDPOINT).credentialsProvider(staticProvider).build();
        List<Bucket> listBuckets = minioClient.listBuckets();
        System.out.println(listBuckets.stream().map(Bucket::name).collect(Collectors.toList()));

        File file = new File("/Users/agefades/Downloads/tx.jpeg");
        //这个objectName的值是经过上面的policy授权的
        String objectName = "mx/pic1.jpg";
        FileInputStream fileInputStream = new FileInputStream(file);
        minioClient.putObject(PutObjectArgs.builder().bucket(BUCKET)
                .object(objectName)
                .contentType("image/jpeg")
                .stream(fileInputStream, fileInputStream.available(), -1).build());

    }

}
```

#### 测试结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1677580860746.png)