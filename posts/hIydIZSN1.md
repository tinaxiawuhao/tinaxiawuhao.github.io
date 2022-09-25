---
title: 'mybatis级联查询'
date: 2021-12-27 11:17:49
tags: [mybatis]
published: true
hideInList: false
feature: /post-images/hIydIZSN1.png
isTop: false
---
## ModelMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.haier.biz.mapper.ModelMapper">
    <resultMap id="BaseResultMap" type="com.haier.biz.entity.Model">
        <id column="model_id" jdbcType="BIGINT" property="modelId"/>
        <result column="equipment_model_mark" jdbcType="VARCHAR" property="equipmentModelMark"/>
        <result column="equipment_name" jdbcType="VARCHAR" property="equipmentName"/>
        <result column="specification_model" jdbcType="VARCHAR" property="specificationModel"/>
        <result column="model_description" jdbcType="VARCHAR" property="modelDescription"/>
        <result column="model_sort_key" jdbcType="VARCHAR" property="modelSortKey"/>
        <result column="model_classification_label" jdbcType="VARCHAR" property="modelClassificationLabel"/>
        <result column="tenant_product_id" jdbcType="BIGINT" property="tenantProductId"/>
        <result column="tenant_product_name" jdbcType="VARCHAR" property="tenantProductName"/>
        <result column="manufacturer" jdbcType="VARCHAR" property="manufacturer"/>
        <result column="create_by" jdbcType="BIGINT" property="createBy"/>
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime"/>
        <result column="update_by" jdbcType="BIGINT" property="updateBy"/>
        <result column="update_time" jdbcType="TIMESTAMP" property="updateTime"/>
    </resultMap>
    <!--级联查询-->
    <resultMap id="BaseResultInstanceMap" type="com.haier.biz.entity.DTO.ModelDTO" extends="BaseResultMap">
        <result column="publishNumber" jdbcType="BIGINT" property="publishNumber"/>
        <result column="customer_id" jdbcType="BIGINT" property="customerId"/>
        <result column="topic" jdbcType="VARCHAR" property="topic"/>
        <!--property来源ModelDTO属性，column来源于selectInstancelList查询-->
        <!--查询单条-->
        <association property="equipmentPicture" column="equipment_model_mark"
                     select="getEquipmentPicture"></association>
        <association property="instanceNumber" column="{equipmentModelMark=equipment_model_mark,customerId=customer_id}"
                     select="getCustomerInstanceNumber"></association>
        <!--查询列表-->
        <collection property="equipmentPictures" column="equipment_model_mark"
                    select="getEquipmentPictures"></collection>
    </resultMap>
    <select id="selectInstancelList" parameterType="com.haier.biz.entity.VO.InstanceSelectVo"
            resultMap="BaseResultInstanceMap">
        SELECT
        distinct model.model_id,
        equipment_model_mark,
        equipment_name,
        specification_model,
        model_description,
        model_sort_key,
        model_classification_label,
        tenant_product_id,
        tenant_product_name,
        manufacturer,
        model.create_by,
        model.create_time,
        model.update_by,
        model.update_time,
        0 as publishNumber,
        relation_model_customer.customer_id,
        relation_model_customer.topic
        FROM
        model
        LEFT JOIN relation_model_customer on relation_model_customer.model_id=model.model_id
        <where>
            <if test="customerId != null">
                and relation_model_customer.customer_id = #{customerId}
            </if>
            <if test="modelSortKey != null and modelSortKey!=''">
                and `model`.model_sort_key = #{modelSortKey}
            </if>
            <if test="equipmentName != null and equipmentName!=''">
                and `model`.equipment_name like CONCAT('%', #{equipmentName,jdbcType=VARCHAR},'%')
            </if>
        </where>
    </select>
    <select id="getEquipmentPicture" parameterType="java.lang.String" resultType="java.lang.String">
        SELECT model_instance_file_associate.file_object_key as equipmentPicture
        from model_instance_file_associate
        where model_instance_file_associate.file_type=4
            and model_instance_file_associate.type=0
            and model_instance_file_associate.del_flag=0
            and model_instance_file_associate.model_or_instance_mark = #{equipmentModelMark,jdbcType=VARCHAR}
        limit 1
    </select>
    <select id="getCustomerInstanceNumber" parameterType="java.util.Map" resultType="java.lang.Long">
        SELECT count(*)
        from model_instance_associate
        left join `instance` on model_instance_associate.instance_mark=`instance`.instance_mark
        where model_instance_associate.equipment_model_mark = #{equipmentModelMark,jdbcType=VARCHAR}
            and `instance`.customer_id=#{customerId}
    </select>
    <select id="getEquipmentPictures" parameterType="java.lang.String" resultType="java.lang.String">
       SELECT model_instance_file_associate.file_object_key as equipmentPictures
        from model_instance_file_associate
        where model_instance_file_associate.file_type=4
            and model_instance_file_associate.type=0
            and model_instance_file_associate.del_flag=0
            and model_instance_file_associate.model_or_instance_mark = #{equipmentModelMark,jdbcType=VARCHAR}
    </select>
</mapper>
```

## ModelMapper

```java
@Repository
public interface ModelMapper extends BaseMapper<Model> {

    List<ModelDTO> selectInstancelList(InstanceSelectVo instanceSelectVo);

    List<String> getEquipmentPictures(@Param("equipmentModelMark") String equipmentModelMark);

}
```

## ModelDTO

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@AllArgsConstructor
@NoArgsConstructor
public class ModelDTO extends Model {

    @ApiModelProperty(value = "产品(模型)图片(取第一张)")
    private String equipmentPicture;

    @ApiModelProperty(value = "产品(模型)图片")
    private List<String> equipmentPictures;

    @ApiModelProperty(value = "发布客户数")
    private Long publishNumber;

    @ApiModelProperty(value = "实例设备数")
    private Long instanceNumber;

    @ApiModelProperty(value = "使用方Id")
    private Long customerId;

    @ApiModelProperty(value = "实例设备运行情况")
    private List<NumberDTO> equipmentList;

    @ApiModelProperty(value = "kafka实时推送的topic")
    private String topic;
}
```