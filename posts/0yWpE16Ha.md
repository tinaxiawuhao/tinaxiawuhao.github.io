---
title: '树形结构查询'
date: 2022-08-25 20:36:06
tags: [mybatis]
published: true
hideInList: false
feature: /post-images/0yWpE16Ha.png
isTop: false
---
树型层级结构查询

### 数据sql

```sql
CREATE TABLE `sys_company` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `code` varchar(36) NOT NULL COMMENT '标识主键',
  `company_name` varchar(255) NOT NULL COMMENT '组织名称',
  `pid` int(10) DEFAULT NULL COMMENT '上级组织id',
  `pname` varchar(255) DEFAULT NULL COMMENT '上级组织名称',
  `sort` int(4) DEFAULT '0' COMMENT '排序',
  `state` int(2) DEFAULT '0' COMMENT '是否可用，0可用，1禁用',
  `create_by` varchar(50) DEFAULT NULL COMMENT '创建者',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` varchar(50) DEFAULT NULL COMMENT '修改者',
  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE,
  KEY `code_index` (`code`) USING BTREE COMMENT '简单索引'
) ENGINE=InnoDB AUTO_INCREMENT=119 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='组织详情'
```

### DTO

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class CompanyDTO {

    @ApiModelProperty(value = "主键")
    private Integer id;

    @ApiModelProperty(value = "标识主键")
    private String code;

    @ApiModelProperty(value = "组织名称")
    private String companyName;

    @ApiModelProperty(value = "上级组织id")
    private Integer pid;

    @ApiModelProperty(value = "上级组织名称")
    private String pname;

    @ApiModelProperty(value = "排序")
    private Integer sort;

    @ApiModelProperty(value = "是否可用，0可用，1禁用")
    private Integer state;

    @ApiModelProperty(value = "创建者")
    private String createBy;

    @ApiModelProperty(value = "创建时间")
    private Date createTime;

    @ApiModelProperty(value = "修改者")
    private String updateBy;

    @ApiModelProperty(value = "修改时间")
    private Date updateTime;

    @ApiModelProperty(value = "子公司组织")
    private List<CompanyDTO> children;

    @ApiModelProperty(value = "编辑删除操作权限标识 true:有 false:无")
    private Boolean modAndDelFlag = false;
    
    private CompanyDTO CompanyDTO;
}

```

### service

```java
public List<CompanyDTO> getCompanyList(Integer companyId) {
    CompanyDTO companyDTO = sysCompanyMapper.queryTopData(companyId);
   
    while (companyDTO.getCompanyDTO()!=null){
        companyDTO = companyDTO.getCompanyDTO();
    }
    List<CompanyDTO> companyDTO = sysCompanyMapper.querySimpleTopData(companyDTO.getId());
}
```

### xml

```xml
	<resultMap id="BaseResultAllMap" type="com.haier.dtosplat.entity.DTO.company.CompanyDTO">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="company_name" property="companyName" />
        <result column="pid" property="pid" />
        <result column="pname" property="pname" />
        <result column="sort" property="sort" />
        <result column="state" property="state" />
        <result column="create_by" property="createBy" />
        <result column="create_time" property="createTime" />
        <result column="update_by" property="updateBy" />
        <result column="update_time" property="updateTime" />
        <association property="companyDTO"
                     column="{pid = pid}"
                     select="querySuper">
        </association>
    </resultMap>

	<resultMap id="BaseResultMap" type="com.haier.dtosplat.entity.DTO.company.CompanyDTO">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="company_name" property="companyName" />
        <result column="pid" property="pid" />
        <result column="pname" property="pname" />
        <result column="sort" property="sort" />
        <result column="state" property="state" />
        <result column="create_by" property="createBy" />
        <result column="create_time" property="createTime" />
        <result column="update_by" property="updateBy" />
        <result column="update_time" property="updateTime" />

        <collection property="children" ofType="com.haier.dtosplat.entity.DTO.company.CompanyDTO"
                    column="{id = id}" select="querySon">
        </collection>
    </resultMap>


    <select id="queryTopData" resultMap="BaseResultAllMap">
        SELECT * FROM sys_company
        WHERE id = #{companyId}
    </select>

    <select id="querySimpleTopData" resultMap="BaseResultMap">
        SELECT * FROM sys_company
        WHERE id = #{companyId}
    </select>

    <select id="querySuper" resultMap="BaseResultMap">
        SELECT * FROM sys_company
        WHERE id = #{pid}
    </select>

    <select id="querySon" resultMap="BaseResultMap">
        SELECT * FROM sys_company
        WHERE pid = #{id}
    </select>
```

