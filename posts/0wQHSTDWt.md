---
title: 'saiku自动对接kylin'
date: 2021-09-28 14:56:08
tags: [olap]
published: true
hideInList: false
feature: /post-images/0wQHSTDWt.png
isTop: false
---
saiku通过添加schema和datasource的形式管理对接入系统的数据源，然后提供界面作为直观的分析数据方式，界面产生mdx，由mondrian连接数据源，解析mdx和执行查询

kylin提供大规模数据的olap能力，通过saiku与kylin的对接，利用saiku的友好界面来很方面的查询

如上的整合，需要手动配置数据源，编写schema的操作，感觉比较繁琐，可以通过修改saiku的代码，到kylin中获取project和cube的各种信息，根据一定规则转换生成schema并作为数据源管理起来，这样就很直接将saiku与kylin无缝对接起来。


### 代码案例

**saiku-wabapp**

```xml
#saiku-beans.properties
sylin.user=ADMIN
sylin.password=ADMIN
sylin.cube.url=http://localhost:7070/kylin/api/cubes
sylin.cubedesc.url=http://localhost:7070/kylin/api/cube_desc
sylin.model.url=http://localhost:7070/kylin/api/model
sylin.url=localhost:7070
```

```java
//saiku-beans.xml
 <bean id="repositoryDsManager" class="org.saiku.service.datasource.RepositoryDatasourceManager" init-method="load" destroy-method="unload">
     <!--aop:scoped-proxy/-->
         ......
         <property name="sylinUser" value="${sylin.user}"/>
         <property name="sylinPassWord" value="${sylin.password}"/>
         <property name="sylinCubeUrl" value="${sylin.cube.url}"/>
         <property name="sylinCubeDescUrl" value="${sylin.cubedesc.url}"/>
         <property name="sylinModelUrl" value="${sylin.model.url}"/>
         <property name="sylinUrl" value="${sylin.url}"/>
 </bean>
```

**saiku-service**

```java
public class RepositoryDatasourceManager implements IDatasourceManager, ApplicationListener<HttpSessionCreatedEvent> {
	private String sylinUser;
    private String sylinPassWord;
    private String sylinCubeUrl;
    private String sylinCubeDescUrl;
    private String sylinModelUrl;
    private String sylinUrl;
    //get.set省略
    ......
    private void loadDatasources(Properties ext) {
        datasources.clear();

        List<DataSource> exporteddatasources = null;

        try {
            String result = execute(sylinCubeUrl);
            JSONArray ja = JSON.parseArray(result);
            for (int i = 0; i < ja.size(); i++) {
                JSONObject js = JSONObject.parseObject(ja.getString(i));
                String newCubeName = js.getString("project") + "#" + js.getString("name");
                datasources.put(js.getString("name"), getSaikuDatasource(newCubeName));
            }
        } catch (Exception e) {
            log.error("Failed add sylin cube to datasource", e);
            e.printStackTrace();
        }

       ......
    }
     private SaikuDatasource getSaikuDatasource(String datasourceName) throws Exception {
        if (datasourceName.contains("#")) {
            String cubeName = datasourceName.split("#")[1].trim();
            String cubeDescString = execute(sylinCubeDescUrl + "/" + cubeName);
            CubeDesc cubeDesc = OBJECT_MAPPER.readValue(JSON.parseArray(cubeDescString).getString(0), CubeDesc.class);
            //CubeDesc cubeDesc = JSONObject.parseObject(JSON.parseArray(cubeDescString).getString(0), CubeDesc.class);
            String modelName = cubeDesc.getModelName();
            String cubeModelString = execute(sylinModelUrl + "/" + modelName);
            DataModelDesc modelDesc = OBJECT_MAPPER.readValue(cubeModelString, DataModelDesc.class);
            //DataModelDesc modelDesc = JSONObject.parseObject(cubeModelString, DataModelDesc.class);
            if (cubeDesc != null && modelDesc != null) {
                addSchema(SchemaUtil.createSchema(datasourceName, cubeDesc, modelDesc), "/datasources/" + datasourceName.replace("#", ".") + ".xml", datasourceName);
            }
            String project = new String();
            if (datasourceName.contains("#")) {
                project = datasourceName.split("#")[0].trim();
            } else {
                project = datasourceName;
            }

            Properties properties = new Properties();
            properties.put("location", "jdbc:mondrian:Jdbc=jdbc:kylin://" + sylinUrl + "/" + project + ";JdbcDrivers=org.apache.kylin.jdbc.Driver" + ";Catalog=mondrian:///datasources/" + datasourceName.replace("#", ".") + ".xml");
            properties.put("driver", "mondrian.olap4j.MondrianOlap4jDriver");
            properties.put("username", sylinUser);
            properties.put("password", sylinPassWord);
            properties.put("security.enabled", false);
            properties.put("advanced", false);
            return new SaikuDatasource(cubeName, SaikuDatasource.Type.OLAP, properties);
        }
        return null;
    }

    private String execute(String url) throws URISyntaxException, IOException {
        int httpConnectionTimeoutMs = 30000;
        int httpSocketTimeoutMs = 120000;
        final HttpParams httpParams = new BasicHttpParams();
        final PoolingClientConnectionManager cm = new PoolingClientConnectionManager();
        HttpConnectionParams.setSoTimeout(httpParams, httpSocketTimeoutMs);
        HttpConnectionParams.setConnectionTimeout(httpParams, httpConnectionTimeoutMs);
        cm.setDefaultMaxPerRoute(20);
        cm.setMaxTotal(200);
        HttpGet get = newGet(url);
        get.setURI(new URI(url));
        DefaultHttpClient client = new DefaultHttpClient(cm, httpParams);
        if (sylinUser != null && sylinPassWord != null) {
            CredentialsProvider provider = new BasicCredentialsProvider();
            UsernamePasswordCredentials credentials = new UsernamePasswordCredentials(sylinUser, sylinPassWord);
            provider.setCredentials(AuthScope.ANY, credentials);
            client.setCredentialsProvider(provider);
        }
        HttpResponse response = client.execute(get);
        if (response.getStatusLine().getStatusCode() != 200) {
            throw new IOException("Invalid response " + response.getStatusLine().getStatusCode());
        }

        String result = getContent(response);
        return result;
    }

    private HttpGet newGet(String url) {
        HttpGet get = new HttpGet();
        addHttpHeaders(get);
        return get;
    }

    private void addHttpHeaders(HttpRequestBase method) {
        method.addHeader("Accept", "application/json, text/plain, */*");
        method.addHeader("Content-Type", "application/json");
        String basicAuth = DatatypeConverter
                .printBase64Binary((sylinUser + ":" + sylinPassWord).getBytes(StandardCharsets.UTF_8));
        method.addHeader("Authorization", "Basic " + basicAuth);
    }

    private String getContent(HttpResponse response) throws IOException {
        InputStreamReader reader = null;
        BufferedReader rd = null;
        StringBuffer result = new StringBuffer();
        try {
            reader = new InputStreamReader(response.getEntity().getContent(), StandardCharsets.UTF_8);
            rd = new BufferedReader(reader);
            String line = null;
            while ((line = rd.readLine()) != null) {
                result.append(line);
            }
        } finally {
            IOUtils.closeQuietly(reader);
            IOUtils.closeQuietly(rd);
        }
        return result.toString();
    }
}
```

**mondrian3.0语法工具类**

```java
package org.saiku.service.util;

import org.apache.kylin.cube.model.CubeDesc;
import org.apache.kylin.cube.model.DimensionDesc;
import org.apache.kylin.metadata.model.*;

import java.util.*;

public class SchemaUtil1 {
    private static String newLine = "\r\n";
    private static Set<String> aggSet = new HashSet<String>(){
        {
            add("sum");
            add("min");
            add("max");
            add("count");
            add("count_distinct");
        }

    };


    public static String createSchema(String dataSourceName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        StringBuffer sb = new StringBuffer();
        sb = appendSchema(sb, dataSourceName, cubeDesc, modelDesc);
        return sb.toString();
    }

    public static StringBuffer appendSchema(StringBuffer sb, String dataSourceName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        sb.append("<?xml version='1.0'?>").append(newLine)
                .append("<Schema name='" + dataSourceName.split("#")[0].trim()+"'>")
                .append(newLine);
        sb = appendCube(sb, dataSourceName, cubeDesc, modelDesc);
        sb.append("</Schema>").append(newLine);
        return sb;
    }

    public static StringBuffer appendCube(StringBuffer sb, String cubeName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        sb.append("<Cube name='" + cubeName.split("#")[1] + "'>").append(newLine);
        sb.append("<Table name='"+dealTableName(modelDesc.getRootFactTableName())+"'/>").append(newLine);
        HashMap<String,String> aliasTableMap = new HashMap<String,String>();
        HashMap<String,String> aliasTableJoinMap = new HashMap<String,String>();
        aliasTableMap.put(dealTableName(modelDesc.getRootFactTableName()),dealTableName(modelDesc.getRootFactTableName()));
        for(JoinTableDesc joinTableDesc:modelDesc.getJoinTables()) {
            aliasTableMap.put(joinTableDesc.getAlias(),dealTableName(joinTableDesc.getTable()));
            if(joinTableDesc.getJoin().getPrimaryKey().length == 1){
                aliasTableJoinMap.put(dealTableName(joinTableDesc.getAlias()),joinTableDesc.getJoin().getPrimaryKey()[0].concat("#").concat(joinTableDesc.getJoin().getForeignKey()[0]));
            }
        }

        for(DimensionDesc dimensionDesc: cubeDesc.getDimensions()){
            // tableSet.add(dealTableName(dimensionDesc.getTable()));
            if(aliasTableMap.get(dealTableName(dimensionDesc.getTable())).equals(dealTableName(modelDesc.getRootFactTableName()))){
                sb.append("<Dimension name='"+dimensionDesc.getName()+"'>").append(newLine);
                sb.append("<Hierarchy name='"+dimensionDesc.getName()+"' hasAll='true' allMemberName='All "+dimensionDesc.getName()+"'>").append(newLine);

            }else if(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable()))==null){
                continue;
            } else if(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable()))!=null && aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[1].split("\\.")[0]
                    .equals(dealTableName(modelDesc.getRootFactTableName()))){
                sb.append("<Dimension name='"+dimensionDesc.getName()+"' foreignKey='"+aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable()))
                        .split("#")[1].split("\\.")[1]+"'>").append(newLine);

                sb.append("<Hierarchy name='"+dimensionDesc.getName()+"' hasAll='true' allMemberName='All "+dimensionDesc.getName()+"' primaryKey='"+
                        aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[0].split("\\.")[1]+"'>").append(newLine);

                sb.append("<Table name='"+aliasTableMap.get(dealTableName(dimensionDesc.getTable()))+"'/>").append(newLine);

            } else if(aliasTableJoinMap.get(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[1].split("\\.")[0])
                    .split("#")[1].split("\\.")[0].equals(dealTableName(modelDesc.getRootFactTableName()))){
                sb.append("<Dimension name='"+dimensionDesc.getName()+"' foreignKey='"+aliasTableJoinMap.get(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable()))
                        .split("#")[1].split("\\.")[0]).split("#")[1].split("\\.")[1]+"'>").append(newLine);

                sb.append("<Hierarchy name='"+dimensionDesc.getName()+"' hasAll='true' allMemberName='All "+dimensionDesc.getName()+"' primaryKey='"+
                        aliasTableJoinMap.get(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[1].split("\\.")[0])
                                .split("#")[0].split("\\.")[1]+"' primaryKeyTable='"+aliasTableMap.get(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable()))
                        .split("#")[1].split("\\.")[0])+"'>").append(newLine);

                sb.append("<Join leftKey='"+aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[1].split("\\.")[1]+"' rightAlias='"+aliasTableMap.get(dimensionDesc.getTable())+
                        "' rightKey='"+aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[0].split("\\.")[1]+"'>").append(newLine);

                sb.append("<Table name='"+aliasTableMap.get(aliasTableJoinMap.get(dealTableName(dimensionDesc.getTable())).split("#")[1].split("\\.")[0])+"'/>").append(newLine);
                sb.append("<Table name='"+aliasTableMap.get(dimensionDesc.getTable())+"'/>").append(newLine);
                sb.append("</Join>").append(newLine);
            } else {
                continue;
            }
            Set<String> columns = getColumns(dimensionDesc);
            for(String column:columns){
                sb.append("<Level name='"+column+"' column='"+column+"' table='"+aliasTableMap.get(dimensionDesc.getTable())+"'/>").append(newLine);
            }
            sb.append("</Hierarchy>").append(newLine);
            sb.append("</Dimension>").append(newLine);
        }
        for (MeasureDesc measureDesc : cubeDesc.getMeasures()) {
            int i=0;
            String table = measureDesc.getFunction().getParameter().getValue().split("\\.")[0];
            final boolean flag = Arrays.stream(modelDesc.getJoinTables()).anyMatch(item -> table.equals(dealTableName(item.getTable())));
            if(table.equals(dealTableName(modelDesc.getRootFactTableName()))||flag||table.equals("1")){
                addMeasure(sb,measureDesc,getColumn(cubeDesc,modelDesc,dealTableName(modelDesc.getRootFactTableName())));
            }
        }

        sb.append("</Cube>").append(newLine);
        return sb;
    }



    public static String dealTableName(String tableName){
        if(tableName.contains("."))
            return tableName.split("\\.")[1];
        else
            return tableName;
    }

    public static StringBuffer addMeasure(StringBuffer sb, MeasureDesc measureDesc, String defaultColumn) {
        FunctionDesc funtionDesc = measureDesc.getFunction();
        String aggregator = funtionDesc.getExpression().trim().toLowerCase();
        if(aggSet.contains(aggregator.toLowerCase())){
            //mondrian only have distinct-count
            if(aggregator.equals("count_distinct")){
                aggregator = "distinct-count";
            }
            if(funtionDesc.getParameter().getValue().equals("1")) {
                sb.append("<Measure aggregator='" + aggregator + "' column='" + defaultColumn + "' name='" + measureDesc.getName() + "' visible='true'/>")
                        .append(newLine);
            }
            else
                sb.append("<Measure aggregator='" + aggregator + "' column='" + funtionDesc.getParameter().getValue().split("\\.")[1] + "' name='" + measureDesc.getName() + "' visible='true'/>")
                        .append(newLine);
            return sb;
        }
        return sb;
    }

    public static String getColumn(CubeDesc cubeDesc,DataModelDesc dataModelDesc,String tableName){
        List<MeasureDesc> measureDescList = cubeDesc.getMeasures();
        for(MeasureDesc measureDesc:measureDescList){
            if(measureDesc.getFunction().getParameter().getValue().split("\\.")[0].equals(tableName)){
                return measureDesc.getFunction().getParameter().getValue().split("\\.")[1];
            }
        }
        if(dataModelDesc.getMetrics().length>0){
            return dataModelDesc.getMetrics()[0];
        }

        for(ModelDimensionDesc modelDimensionDesc:dataModelDesc.getDimensions()){
            if(modelDimensionDesc.getTable().equals(tableName)){
                return modelDimensionDesc.getColumns()[0];
            }
        }
        return null;
    }

    public static Set<String> getColumns(DimensionDesc dimensionDesc){
        Set<String> columns = new HashSet<String>();
        if (dimensionDesc.getColumn() != null || dimensionDesc.getDerived() != null) {
            if(dimensionDesc.getColumn() != null) {
                columns.add(dimensionDesc.getColumn());
            }
            if (dimensionDesc.getDerived() != null) {
                for (String derived : dimensionDesc.getDerived()) {
                    columns.add(derived);
                }
            }
        } else {
            columns.add(dimensionDesc.getName());
        }
        return columns;
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Schema name="Mondrian">  <!--模型定义-->
<Cube name="Person">     <!--立方体 ，一个立方体有多个维度-->
    <Table name="PERSON" />  <!--立方体对应的表 -->
     <Dimension name="部门" foreignKey="USERID" > <!--定义维度 -->
        <Hierarchy  hasAll="true" primaryKey="USERID" allMemberName="所有部门" > <!--定义维度下面的层次，层次包含很多层 -->
          <Table name="PERSON" alias="a"/>                                   <!--定义维度获取数据的来源-表 -->
        <Level name="部门" column="DEPARTMENT" uniqueMembers="true" /> <!--定义层次的层，每个层对应数据库中对应的字段 -->
        <Level name="姓名" column="USERNAME" uniqueMembers="true" />           
        </Hierarchy>
    </Dimension>     
    <Dimension name="性别"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有性别">         
            <Table name="PERSON" alias="b" />
        <Level name="性别" column="SEX"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>
    <Dimension name="专业技术资格类别"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有专业技术资格类别">         
            <Table name="PERSON" alias="c" />
        <Level name="资格类别" column="ZYJSLB"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>
    <Dimension name="专业技术资格等级"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有专业技术资格等级">         
            <Table name="PERSON" alias="d" />
        <Level name="资格等级" column="ZYJSDJ"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>
     <Dimension name="职系"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有职系">         
            <Table name="PERSON" alias="e" />
        <Level name="职系" column="ZHIXI"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>
    <Dimension name="民族"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有民族">         
            <Table name="PERSON" alias="f" />
        <Level name="民族" column="NATIONALITY"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>
    <Dimension name="学历"  foreignKey="USERID" >
        <Hierarchy hasAll="true" primaryKey="USERID" allMemberName="所有学历">         
            <Table name="PERSON" alias="g" />
        <Level name="学历" column="XUELI"   uniqueMembers="true" />
        </Hierarchy>
    </Dimension>       
    <Measure name="人数" column="USERID" aggregator="distinct count" />       <!--指标/度量，采用distinct count聚合 -->
    </Cube>
</Schema>
```



**mondrian4.0语法工具类**

```java
package org.saiku.service.util;

import org.apache.kylin.cube.model.CubeDesc;
import org.apache.kylin.cube.model.DimensionDesc;
import org.apache.kylin.cube.model.RowKeyDesc;
import org.apache.kylin.metadata.model.*;

import java.util.*;

public class SchemaUtil {
    private static final String newLine = "\r\n";
    private static Map<String, String> map;

    public static String createSchema(String dataSourceName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        StringBuffer sb = new StringBuffer();
        sb = appendSchema(sb, dataSourceName, cubeDesc, modelDesc);
//        System.out.println("********************************" + sb.toString());
        return sb.toString();
    }

    public static StringBuffer appendSchema(StringBuffer sb, String dataSourceName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        sb.append("<?xml version='1.0'?>").append(newLine)
                .append("<Schema name='").append(dataSourceName.split("#")[0].trim()).append("' metamodelVersion='4.0'>")
//                .append("<Schema name='" + dataSourceName + "' metamodelVersion='4.0'>")
                .append(newLine);
        appendTable(sb, modelDesc);
        appendDimension(sb, modelDesc);
        appendCube(sb, dataSourceName, cubeDesc, modelDesc);
        sb.append("</Schema>").append(newLine);
        return sb;
    }

    public static StringBuffer appendTable(StringBuffer sb, DataModelDesc modelDesc) {
        StringBuilder linkSb = new StringBuilder();
        //获取所有关系表
        final List<ModelDimensionDesc> tables = modelDesc.getDimensions();
        sb.append("<PhysicalSchema>").append(newLine);
        //添加表关联关系
        final JoinTableDesc[] joinTables = modelDesc.getJoinTables();
        for (JoinTableDesc joinTableDesc : joinTables) {
            String factTablename = dealModelTableName(joinTableDesc.getJoin().getForeignKey()[0]);
            String Column = dealTableName(joinTableDesc.getJoin().getForeignKey()[0]);
            String joinTablename = dealModelTableName(joinTableDesc.getJoin().getPrimaryKey()[0]);
            String joinColumn = dealTableName(joinTableDesc.getJoin().getPrimaryKey()[0]);
            linkSb.append("<Link source='").append(factTablename).append("' target='").append(joinTablename).append("'>").append(newLine)
                    .append("<ForeignKey>").append(newLine)
                    .append("<Column name='").append(Column).append("'/>").append(newLine)
                    .append("</ForeignKey>").append(newLine)
                    .append("</Link>").append(newLine);
            //清空map
            map = new HashMap<>();
            tables.forEach(item -> {
                if (item.getTable().equals(factTablename)) {
                    map.put(factTablename, Column);
                }
                if (item.getTable().equals(joinTablename)) {
                    map.put(joinTablename, joinColumn);
                }
            });
        }
        //添加表
        for (String tableName : map.keySet()) {
            sb.append("<Table name='").append(tableName).append("'>").append(newLine)
                    .append("<Key>").append(newLine).append("<Column name='").append(map.get(tableName)).append("'/>").append(newLine)
                    .append("</Key>").append(newLine)
                    .append("</Table>").append(newLine);
        }

        sb.append(linkSb);
        linkSb.delete(0, linkSb.length());
        sb.append("</PhysicalSchema>").append(newLine);
        return sb;
    }

    public static Map<String, JoinDesc> getJoinDesc(DataModelDesc modelDesc) {
        Map<String, JoinDesc> joinDescMap = new HashMap<String, JoinDesc>();
        for (JoinTableDesc lookupDesc : modelDesc.getJoinTables()) {
            if (!joinDescMap.containsKey(dealTableName(lookupDesc.getTable())))
                joinDescMap.put(dealTableName(lookupDesc.getTable()), lookupDesc.getJoin());
        }
        return joinDescMap;
    }

    public static void appendDimension(StringBuffer sb, DataModelDesc modelDesc) {
        StringBuilder hierSb = new StringBuilder();
        for (ModelDimensionDesc dimensionDesc : modelDesc.getDimensions()) {
            sb.append("<Dimension name='").append(dimensionDesc.getTable()).append("' key='").append(map.get(dimensionDesc.getTable())).append("' table='").append(dimensionDesc.getTable()).append("'>").append(newLine);
            sb.append("<Attributes>").append(newLine);
            hierSb.append("<Hierarchies>").append(newLine);
            hierSb.append("<Hierarchy  name='").append(dimensionDesc.getTable()).append("' hasAll='true'>").append(newLine);
            for (String column : dimensionDesc.getColumns()) {
                // add Attributes to stringbuffer
                addAttribute(sb, column);
                addHierarchy(hierSb, column);
            }
            sb.append("</Attributes>").append(newLine);
            hierSb.append("</Hierarchy>").append(newLine);
            hierSb.append("</Hierarchies>").append(newLine);
            sb.append(hierSb);
            hierSb.delete(0, hierSb.length());
            sb.append("</Dimension>").append(newLine);
        }
    }

    public static Set<String> getColumns(DimensionDesc dimensionDesc) {
        Set<String> columns = new HashSet<String>();
        if (dimensionDesc.getColumn() != null || dimensionDesc.getDerived() != null) {
            if (dimensionDesc.getColumn() != null) {
//                for (String column : dimensionDesc.getColumn()) {
                columns.add(dimensionDesc.getColumn());
//                }
            }
            if (dimensionDesc.getDerived() != null) {
                columns.addAll(Arrays.asList(dimensionDesc.getDerived()));
            }
        } else {
            columns.add(dimensionDesc.getName());
        }
        return columns;
    }

    public static StringBuffer addAttribute(StringBuffer sb, String attr) {
        sb.append("<Attribute name='").append(attr).append("' keyColumn='").append(attr).append("' hasHierarchy='false'/>").append(newLine);
        return sb;
    }

    public static void addHierarchy(StringBuilder sb, String attr) {
        sb.append("<Level attribute='").append(attr).append("'/>").append(newLine);
    }


    public static String dealTableName(String tableName) {
        if (tableName.contains("."))
            return tableName.split("\\.")[1];
        else
            return tableName;
    }

    public static String dealModelTableName(String tableName) {
        if (tableName.contains("."))
            return tableName.split("\\.")[0];
        else
            return tableName;
    }

    public static StringBuffer appendCube(StringBuffer sb, String cubeName, CubeDesc cubeDesc, DataModelDesc modelDesc) {
        sb.append("<Cube name='").append(cubeName.split("#")[1].trim()).append("'>").append(newLine);
        addCubeDimension(sb, modelDesc.getDimensions());
        sb.append("<MeasureGroups>").append(newLine);

        Map<String, Map<String, MeasureDesc>> allMap = new HashMap();

        MeasureDesc one = new MeasureDesc();
        for (MeasureDesc measureDesc : cubeDesc.getMeasures()) {
            final String tableName = dealModelTableName(measureDesc.getFunction().getParameter().getValue());
            final String columnName = dealTableName(measureDesc.getFunction().getParameter().getValue());
            if ("1".equals(tableName)) {
                one = measureDesc;
            } else {
                if (Objects.isNull(allMap.get(tableName))) {
                    Map<String, MeasureDesc> map = new HashMap();
                    if (Objects.nonNull(one)) {
                        map.put(columnName, one);
                        one = null;
                    }
                    map.put(columnName, measureDesc);
                    allMap.put(tableName, map);
                } else {
                    allMap.get(tableName).put(columnName, measureDesc);
                }
            }
        }

        allMap.forEach((tableName, value) -> {
            sb.append("<MeasureGroup table='").append(dealTableName(tableName)).append("'>").append(newLine);
            addDimensionLink(sb, modelDesc);
            sb.append("<Measures>").append(newLine);
            value.forEach((columnName, measureDesc) -> {
                addMeasure(sb, measureDesc, getColumn(cubeDesc));
            });
            sb.append("</Measures>").append(newLine);
            sb.append("</MeasureGroup>").append(newLine);
        });
        sb.append("</MeasureGroups>").append(newLine);
        sb.append("</Cube>").append(newLine);
        return sb;
    }

    public static void addCubeDimension(StringBuffer sb, List<ModelDimensionDesc> dimensionDescs) {
        sb.append("<Dimensions>").append(newLine);
        for (ModelDimensionDesc dimensionDesc : dimensionDescs) {
            sb.append("<Dimension source='").append(dimensionDesc.getTable()).append("' visible='true'/>").append(newLine);
        }
        sb.append("</Dimensions>").append(newLine);
    }

    public static void addDimensionLink(StringBuffer sb, DataModelDesc modelDesc) {
        sb.append("<DimensionLinks>").append(newLine);
        for(ModelDimensionDesc dimensionDesc : modelDesc.getDimensions()) {
            if(dimensionDesc.getTable().contains(dealTableName(modelDesc.getRootFactTableName()))) {
                sb.append("<FactLink dimension='").append(dimensionDesc.getTable()).append("'/>").append(newLine);
            }else{
                sb.append("<ForeignKeyLink dimension='").append(dimensionDesc.getTable()).append("' foreignKeyColumn='").append(map.get(dimensionDesc.getTable())).append("'/>").append(newLine);
            }
        }
        sb.append(" </DimensionLinks>").append(newLine);
    }

    public static StringBuffer addMeasure(StringBuffer sb, MeasureDesc measureDesc, String defaultColumn) {
        FunctionDesc funtionDesc = measureDesc.getFunction();
        String aggregator = funtionDesc.getExpression().trim().toLowerCase();
        //mondrian only have distinct-count
        if (aggregator.equals("count_distinct")) {
            aggregator = "distinct-count";
        }
        if (funtionDesc.getParameter().getValue().equals("1")) {
            sb.append("<Measure aggregator='").append(aggregator).append("' column='").append(dealTableName(defaultColumn)).append("' name='").append(measureDesc.getName()).append("' visible='true'/>")
                    .append(newLine);
        } else
            sb.append("<Measure aggregator='").append(aggregator).append("' column='").append(dealTableName(funtionDesc.getParameter().getValue())).append("' name='").append(measureDesc.getName()).append("' visible='true'/>")
                    .append(newLine);
        return sb;
    }

    public static Set<String> getTables(List<DimensionDesc> dimensionDescList) {
        Set<String> tables = new HashSet<String>();
        for (DimensionDesc dimensionDesc : dimensionDescList) {
            String table = dealTableName(dimensionDesc.getTable());
            tables.add(table);
        }
        return tables;
    }

    public static String getColumn(CubeDesc cubeDesc) {
        RowKeyDesc rowKey = cubeDesc.getRowkey();
        return rowKey.getRowKeyColumns()[0].getColumn();
    }
}
```

```xml
<?xml version='1.0'?>
<Schema name='xiaowei' metamodelVersion='4.0'>
<PhysicalSchema>
	<Table name='TBL_FARM_INCOME_STATICS'>
		<Key>
			<Column name='COMPANY_ID'/>
		</Key>
	</Table>
	<Table name='TBL_CUSTOMER'>
		<Key>
			<Column name='COMPANY_ID'/>
		</Key>
	</Table>
	<Link source='TBL_FARM_INCOME_STATICS' target='TBL_CUSTOMER'>
		<ForeignKey>
			<Column name='COMPANY_ID'/>
		</ForeignKey>
	</Link>
</PhysicalSchema>
<Dimension name='TBL_FARM_INCOME_STATICS' table='TBL_FARM_INCOME_STATICS' key='COMPANY_ID'>
	<Attributes>
		<Attribute name='COMPANY_ID' keyColumn='COMPANY_ID' hasHierarchy='false'/>
		<Attribute name='INCOME_DATE' keyColumn='INCOME_DATE' hasHierarchy='false'/>
	</Attributes>

	<Hierarchies>
		<Hierarchy name='TBL_FARM_INCOME_STATICS' allMemberName='All Warehouses'>
			<Level attribute='COMPANY_ID'/>
			<Level attribute='INCOME_DATE'/>
		</Hierarchy>
	</Hierarchies>
</Dimension>

<Dimension name='TBL_CUSTOMER' table='TBL_CUSTOMER' key='COMPANY_ID'>
	<Attributes>
		<Attribute name='COMPANY_ID' keyColumn='COMPANY_ID' hasHierarchy='false'/>
		<Attribute name='CUSTOMER_TYPE' keyColumn='CUSTOMER_TYPE' hasHierarchy='false'/>
		<Attribute name='PHONE' keyColumn='PHONE' hasHierarchy='false'/>
	</Attributes>

	<Hierarchies>
		<Hierarchy name='TBL_CUSTOMER' allMemberName='All Warehouses'>
			<Level attribute='COMPANY_ID'/>
			<Level attribute='CUSTOMER_TYPE'/>
			<Level attribute='PHONE'/>
		</Hierarchy>
	</Hierarchies>
</Dimension>

<Cube name='tbl_farm_income_statics'>
<Dimensions>
<Dimension source='TBL_FARM_INCOME_STATICS' visible='true'/>
<Dimension source='TBL_CUSTOMER' visible='true'/>
</Dimensions>
<MeasureGroups>
	<MeasureGroup table='TBL_CUSTOMER'>
		<Measures>
			<Measure aggregator='sum' column='CONSUME_AMMOUT' name='CONSUME_AMMOUT_SUM' visible='true'/>
		</Measures>
		<DimensionLinks>
			<FactLink dimension='TBL_FARM_INCOME_STATICS'/>
			<ForeignKeyLink dimension='TBL_CUSTOMER' foreignKeyColumn='COMPANY_ID'/>
		</DimensionLinks>
	</MeasureGroup>

	<MeasureGroup table='TBL_FARM_INCOME_STATICS'>
		<Measures>
			<Measure aggregator='count' column='COMPANY_ID' name='_COUNT_' visible='true'/>
			<Measure aggregator='sum' column='REFUND_AMOUNT' name='REFUND_AMOUNT_SUM' visible='true'/>
			<Measure aggregator='sum' column='COMMON_REFUND_AMOUNT' name='COMMON_REFUND_AMOUNT_SUM' visible='true'/>
			<Measure aggregator='sum' column='COMMON_INCOME' name='COMMON_INCOME_SUM' visible='true'/>
			<Measure aggregator='sum' column='SALE_AMOUNT' name='SALE_AMOUNT_SUM' visible='true'/>
			<Measure aggregator='sum' column='RENEW_AMOUNT' name='RENEW_AMOUNT_SUM' visible='true'/>
			<Measure aggregator='sum' column='TOTAL_AMOUNT' name='TOTAL_AMOUNT_SUM' visible='true'/>
		</Measures>
		<DimensionLinks>
			<FactLink dimension='TBL_FARM_INCOME_STATICS'/>
			<ForeignKeyLink dimension='TBL_CUSTOMER' foreignKeyColumn='COMPANY_ID'/>
		</DimensionLinks>
	</MeasureGroup>
</MeasureGroups>
</Cube>
</Schema>

```

