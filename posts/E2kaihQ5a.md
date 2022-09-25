---
title: 'jar包内类动态加载'
date: 2022-01-04 11:13:15
tags: [java]
published: true
hideInList: false
feature: /post-images/E2kaihQ5a.png
isTop: false
---


### 实体对象

```java
public class CmptDef {
    private String idCmptDef; //组件在DB里的唯一ID号
    private String name;//组件的唯一名称，只能是字母、数字、下划线组成，大于1小雨50，通常为Jar的名字
    private CmptCategory cmptCategory; //种类
    private String fullQualifiedName; //组件入口类全限定名
    private String extraParas; //组件额外参数
    private String formUrl; //组件自定义表单
    private Float version = 1.0f; //组件版本
    private CmptExecutePos executePos = CmptExecutePos.PRE; //组件在网关中执行的位置
    private Integer priority = 100; //组件在相同位置的执行优先级，数字越小越高
    private Integer timeout = 1000;//组件执行超时时间，单位毫秒
    private String description; //组件描述
    private Boolean defaultVersion; //是否是默认有效版本
    private CmptStatus cmptStatus = CmptStatus.editing; // 组件状态
    private String remarks; // 组件发布时需要填写备注信息
    private String code; // 组件code，同一个组件的多个版本的code是一样的
    private Date releaseTime; // 组件发布时间
    private CustomFormCode customFormCode; // 自定义表单里类型
    private String cmptType; //组件类型（特殊组件|普通组件|默认组件）
}
/**
 * 组件类型。
 */
public enum CmptCategory {
    AUTHENTICATION, //认证
    AUTHORIZATION, //鉴权
    FLOW_MANAGEMENT, //流量管理
    REQUEST_COUNT_MANAGEMENT, //请求次数管理
    CACHE, //缓存
    ROUTER, //路由
    TRANSFORM, //数据转换
    LOGGER, //日志
    OTHER//其它
}
/**
 * 组件执行的位置。
 */
public enum CmptExecutePos {
    PRE, //调用上游服务前
    ROUTING, //调用中
    AFTER //调用后
}
public enum CmptStatus {
    editing, //编辑中
    published, //已经发布
    offline //已经下线
}
/**
 * 组件风格。
 */
public enum CustomFormCode {
    RESTFUL_FORM,
    SQL_FORM,
    DUBBO_FORM,
    OTHER,
    MQ_FORM,
    WEBSERVICE_FORM
}
```

### 方法接口

```java
public interface ICmptService {
    /**
     * 根据组件类名和版本从缓存中获取组件,如果不存在则尝试动态加载组件类，实例化并刷新缓存后再获取，还是不存在则返回null；
     * 另外配置更新会有单独的线程刷新缓存。
     *
     * @return
     */
    ICmpt getCmptInstance(final CmptDef cmptDef);

    /**
     * 刷新API关联的组件配置信息
     *
     * @param apis
     * @param ignoreRefreshTime
     * @return
     */
    boolean refreshCmptInstanceCache(List<Api> apis, boolean ignoreRefreshTime);

    /**
     * 删除组件实例缓存
     *
     * @param fullQualifiedName
     * @param code
     */
    void removeCmpt(final String fullQualifiedName, final Float version, final String code);
}
```

### 方法实现

```java
@Service
public class CmptServiceImpl implements ICmptService {
    private static Logger logger = LoggerFactory.getLogger(CmptServiceImpl.class);

    @Value("${cmpt.dynamicLoadCmptClass}")
    private boolean dynamicLoadCmptClass;

    @Autowired
    private ICmptDefService cmptDefService;

    @Override
    public synchronized ICmpt getCmptInstance(final CmptDef cmptDef) {
        ICmpt cmpt = CmptInstanceHolder.getInstance().getCmpt(cmptDef.getFullQualifiedName(), cmptDef.getVersion(), cmptDef.getCode());
        if (null == cmpt) {
            if (logger.isDebugEnabled()) {
                logger.debug("反射拿到实例>>> " + cmptDef.getFullQualifiedName());
            }
            if (this.dynamicLoadCmptClass) {
                cmpt = CmptClassLoaderUtil.newInstance(cmptDef);
            } else {
                try {
                    Class clazz = Class.forName(cmptDef.getFullQualifiedName());
                    cmpt = (ICmpt) clazz.newInstance();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            if (null != cmpt) {
                if (cmpt instanceof AbstractCmpt) {
                    //设置版本号
                    ((AbstractCmpt) cmpt).setVersion(cmptDef.getVersion());
                    ((AbstractCmpt) cmpt).setIdCmptDef(cmptDef.getIdCmptDef());
                }
                CmptInstanceHolder.getInstance().addEntry(cmpt, cmptDef.getFullQualifiedName(), cmptDef.getVersion(), cmptDef.getCode());
            }
        } else {
            if (logger.isDebugEnabled()) {
                logger.debug("缓存拿到实例>>> " + cmptDef.getFullQualifiedName());
            }
        }
        if (null != cmpt) {
            ApplicationContext context = SpringContextHolder.getContext();
            if (null != context) {
                ZkClient zkClient = context.getBean(ZkClient.class);
                if (null != zkClient) {
                    BaseListener listener = zkClient.getListener();
                    listener.addObserver((AbstractCmpt) cmpt);
                }
            }
        }
        return cmpt;
    }

    @Override
    public boolean refreshCmptInstanceCache(List<Api> apis, boolean ignoreRefreshTime) {
        return false;
    }

    @Override
    public synchronized void removeCmpt(final String fullQualifiedName, final Float version, final String code) {
        ICmpt cmpt = CmptInstanceHolder.getInstance().getCmpt(fullQualifiedName, version, code);
        if (null != cmpt) {
            ApplicationContext context = SpringContextHolder.getContext();
            if (null != context) {
                ZkClient zkClient = context.getBean(ZkClient.class);
                if (null != zkClient) {
                    BaseListener listener = zkClient.getListener();
                    listener.deleteObserver((AbstractCmpt) cmpt);
                }
            }
            cmpt.destroy();
            CmptDef cmptDef = CmptDefHolder.getInstance().getCmptDef(fullQualifiedName, version, code);
            if (null == cmptDef) {
                cmptDef = new CmptDef();
                cmptDef.setFullQualifiedName(fullQualifiedName);
                cmptDef.setVersion(version);
                cmptDef.setCode(code);
            }
            //删除引用实体
            CmptInstanceHolder.getInstance().removeEntry(fullQualifiedName, version, code);
            cmpt = null;

            String jarPath = CmptClassLoaderUtil.getJarPath(cmptDef);
            if (logger.isDebugEnabled()) {
                logger.debug("尝试卸载jar包: " + jarPath);
            }
            CmptClassLoaderManager.unLoadJar(jarPath);
        }
        CmptInstanceHolder.getInstance().removeEntry(fullQualifiedName, version, code);
    }
}
```

### 工具类

```java
/**
 * 组件配置缓存持有者
 */
public class CmptInstanceHolder {
    private static CmptInstanceHolder cmptInstanceHolder = new CmptInstanceHolder();
    //API所关联的组件配置缓存 key: fullQualifiedName    value: ICmpt
    private Map<String, ICmpt> cmpts = new ConcurrentHashMap<>();

    private CmptInstanceHolder() {
    }

    public static CmptInstanceHolder getInstance() {
        return CmptInstanceHolder.cmptInstanceHolder;
    }

    /**
     * 根据类名加版本从缓存中获取组件实例,如果不存在直接返回null；
     *
     * @return
     */
    public ICmpt getCmpt(final String fullQualifiedName, final Float version, String code) {
        final String key = buildKey(fullQualifiedName, version, code);
        return this.cmpts.get(key);
    }

    /**
     * 添加对象
     */
    public void addEntry(final ICmpt cmpt, final String fullQualifiedName, final Float version, final String code) {
        if (cmpt == null || StringUtils.isEmpty(fullQualifiedName) || StringUtils.isEmpty(code)) {
            return;
        }
        final String key = buildKey(fullQualifiedName, version, code);
        removeEntry(fullQualifiedName, version, code);
        this.cmpts.put(key, cmpt);
    }

    private String buildKey(final String fullQualifiedName, Float version, final String code) {
        return fullQualifiedName + "." + version + code;
    }

    public void removeEntry(final String fullQualifiedName, final Float version, final String code) {
        ICmpt remove = this.cmpts.remove(buildKey(fullQualifiedName, version, code));
        if (null != remove) {
            remove.destroy();
        }
    }

    /**
     * 获取所有的组件实例
     * @return
     */
    public Map<String, ICmpt> getAllCmpts(){
        return this.cmpts;
    }
}
```

```java
public class CmptClassLoaderUtil {

    private static Logger logger = LoggerFactory.getLogger(CmptClassLoaderUtil.class);

    public static ICmpt newInstance(final CmptDef cmptDef) {
        ICmpt cmpt = null;
        try {
            final String jarPath = getJarPath(cmptDef);
            logger.info("尝试载入jar包,jar包路径: " + jarPath);
            //加载依赖jar
            CmptClassLoader cmptClassLoader = CmptClassLoaderManager.loadJar(cmptDef.getIdCmptDef(), jarPath, true);
            // 创建实例
            if (null != cmptClassLoader) {
                cmpt = LoadClassUtil.newObject(cmptDef, ICmpt.class, cmptClassLoader);
            } else {
                logger.error("加载组件jar包失败! jarPath: " + jarPath);
            }
        } catch (Exception e) {
            logger.error("组件类加载失败，请检查类名和版本是否正确。ClassName=" + cmptDef.getFullQualifiedName() + ", Version=" + cmptDef.getVersion());
            e.printStackTrace();
        }
        return cmpt;
    }

    public static String getJarPath(final CmptDef cmptDef) {
        StringBuffer sb = new StringBuffer();
        sb.append(AppConfigUtil.getValue("app.home"));
        sb.append(AppConfigUtil.getValue("cmpt.location"));
        sb.append("/");
        //开发中关闭多级目录,不好部署
        sb.append(cmptDef.getCode());
        sb.append("/");
        sb.append(cmptDef.getVersion());
        sb.append("/");
        String[] split = cmptDef.getFullQualifiedName().split("\\.");
        sb.append(split[split.length - 1]);
        sb.append("_");
        sb.append(cmptDef.getVersion());
        sb.append(".jar");
        if (logger.isDebugEnabled()) {
            logger.debug("构建jar包路径: " + sb.toString());
        }
        return sb.toString();
    }

}
```

```shell
#linux版home目录
app.home=/work/sharestore
#组件资源存放路径，<home>/cmpt/<code>/<version>/<name>_<version>.jar,html
cmpt.location=/cmpt
```

```java
/**
 * 类加载器管理, 加载, 卸载jar包
 */
public class CmptClassLoaderManager {

    private static final Logger logger = Logger.getLogger(CmptClassLoaderManager.class);

    private static Map<String, CmptClassLoader> classLoaderMap = new ConcurrentHashMap<>();

    private CmptClassLoaderManager() {
    }

    /**
     * 载入Jar包, 判断是否重新载入Jar包
     *
     * @param fileName
     * @param isReloadJar
     * @return
     */
    public static CmptClassLoader loadJar(String IdCmptDef, String fileName, boolean isReloadJar) {
        fileName = FileUtil.fixFileName(fileName);
        CmptClassLoader cmptClassLoader = classLoaderMap.get(IdCmptDef);
        if (isReloadJar || null == cmptClassLoader) {
            if (logger.isDebugEnabled()) {
                logger.debug("从文件载入jar组件");
            }
            return loadJar(IdCmptDef, fileName);
        } else {
            if (logger.isDebugEnabled()) {
                logger.debug("从缓存载入jar组件");
            }
            return cmptClassLoader;
        }
    }

    /**
     * 载入Jar包, 以新Jar包载入
     *
     * @param fileName
     * @return 返回一个类加载器
     */
    private static CmptClassLoader loadJar(String IdCmptDef, String fileName) {
        CmptClassLoader loader;
        try {
            boolean exists = new File(fileName).exists();
            if (exists) {
                if (null == classLoaderMap.get(IdCmptDef) || unLoadJar(IdCmptDef)) {
                    loader = new CmptClassLoader();
                    boolean loadJar = loader.addURL(fileName);
                    if (loadJar) {
                        classLoaderMap.put(IdCmptDef, loader);
                        return loader;
                    } else {
                        loader.close();
                    }
                }
            } else {
                throw new IllegalArgumentException("传入参数错误,文件不存在! file: " + fileName);
            }
        } catch (Exception e) {
            logger.error("",e);
        }
        return null;
    }

    public static boolean unLoadJar(String IdCmptDef) {
        boolean unLoadJar = false;
        if (logger.isDebugEnabled()) {
            logger.debug("请求卸载jar包: " + IdCmptDef);
        }
        try {
            CmptClassLoader loader = classLoaderMap.remove(IdCmptDef);
            if (null != loader) {
                loader.close();
            }
            unLoadJar = true;
        } catch (Exception e) {
            logger.error("",e);
        }
        return unLoadJar;
    }

    /**
     * 获取组件定义对应的实例的类加载器
     *
     * @param idCmptDef
     * @return
     */
    public static ClassLoader getCmptClassLoader(String idCmptDef) {
        return classLoaderMap.get(idCmptDef);
    }
}
```

```java
public class FileUtil {
	public static String fixFileName(String fileName) {
        if (null == fileName) fileName = "";
        fileName = fileName.replaceAll("\\\\", "/");
        fileName = fileName.replaceAll("//", "/");
        return fileName;
    }
}
```

```java
/**
 * 类加载器终极版, 加载, 卸载jar包
 */
public class CmptClassLoader extends URLClassLoader {

    private static final Logger logger = Logger.getLogger(CmptClassLoader.class);

    CmptClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }

    CmptClassLoader(URL[] urls) {
        super(urls);
    }

    CmptClassLoader(URL[] urls, ClassLoader parent, URLStreamHandlerFactory factory) {
        super(urls, parent, factory);
    }

    CmptClassLoader() {
        super(new URL[]{}, findParentClassLoader());
    }

    /**
     * 载入Jar
     *
     * @param fileName
     * @return
     */
    boolean addURL(String fileName) {
        try {
            URL url = new File(fileName).toURL();
            URLClassPath ucp = this.getUCP();
            ucp.addURL(url);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }

    /**
     * 定位当前父类加载器
     *
     * @return
     */
    private static ClassLoader findParentClassLoader() {
        ClassLoader parent = logger.getClass().getClassLoader();
        if (parent == null) {
            throw new RuntimeException("无法获取当前父加载器!");
        }
        return parent;
    }

    private URLClassPath getUCP() {
        URLClassPath ucp = null;
        try {
            Field declaredField = URLClassLoader.class.getDeclaredField("ucp");
            declaredField.setAccessible(true);
            Object o = declaredField.get(this);
            ucp = (URLClassPath) o;
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
        return ucp;
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        this.close();
    }
}
```

```java
/**
 * 类加载器工具类
 */
public class LoadClassUtil {
    private final static LoadClassUtil LOAD_JAR_UTIL = new LoadClassUtil();
    private static Logger logger = LoggerFactory.getLogger(LoadClassUtil.class);

    private LoadClassUtil() {
    }

    /**
     * 载入jar包
     * 将jar包路径添加到系统类加载器扫描类和资源的文件列表里
     *
     * @param fileName jar绝对路径
     * @return
     */
    public static boolean loadJar(String fileName) {
        try {
            if (strNotNull(fileName) && fileExists(fileName)) {//(ClassLoader要与当前程序同一个loader)
                getMethod().invoke(LOAD_JAR_UTIL.getClass().getClassLoader(), getURL(fileName));//添加路径URL
                //getMethod().invoke(Launcher.getLauncher().getClassLoader(), getURL(fileName));//添加路径URL
                return true;
            } else {
                throw new IllegalArgumentException("传入参数错误,文件或不存在! file: " + fileName);
            }
        } catch (Exception e) {
            logger.error("",e);
        }
        return false;
    }

    /**
     * 判断字符串不为空
     *
     * @param str
     * @return
     */
    private static boolean strNotNull(String str) {
        return null != str && !str.equals("");
    }

    private static boolean fileExists(String fileName) {
        return new File(fileName).exists();
    }

    /**
     * 拿到添加扫描类和资源的路径URL的方法
     *
     * @return
     * @throws NoSuchMethodException
     */
    private static Method getMethod() throws NoSuchMethodException {
        Method method = URLClassLoader.class.getDeclaredMethod("addURL", URL.class);
        // 破解方法的访问权限
        method.setAccessible(true);
        return method;
    }

    /**
     * 得到一个URL
     *
     * @param fileName
     * @return
     * @throws MalformedURLException
     */
    private static URL getURL(String fileName) throws MalformedURLException {
        return new File(fileName).toURI().toURL();
    }

    /**
     * 创建对象实例
     *
     * @param className 全限定名
     * @return
     */
    public static Object newObject(String className) {
        try {
            return Class.forName(className).newInstance();
        } catch (Exception e) {
            logger.error("",e);
        }
        return null;
    }

    /**
     * 创建转换类型后的对象实例
     *
     * @param className 全限定名
     * @param tClass    返回类型
     * @param <T>
     * @return
     */
    public static <T> T newObject(String className, Class<T> tClass) {
        try {
            return (T) Class.forName(className).newInstance();
        } catch (Exception e) {
            logger.error("",e);
        }
        return null;
    }

    /**
     * 创建转换类型后的对象实例
     *
     * @param cmptDef 组件定义->全限定名
     * @param tClass    返回类型
     * @param loader    类加载器
     * @param <T>
     * @return
     */
    public static <T> T newObject(CmptDef cmptDef, Class<T> tClass, ClassLoader loader) {
        try {
            String className = cmptDef.getFullQualifiedName();
            Object newInstance = Class.forName(className, true, loader).newInstance();
            return (T) newInstance;
        } catch (Exception e) {
            String jarPath = CmptClassLoaderUtil.getJarPath(cmptDef);
            logger.error("创建组件实例失败! className=" + cmptDef.getFullQualifiedName() + " jarPath=" + jarPath);
            logger.error("创建出错组件:" + cmptDef.getName() + cmptDef.getVersion() + " ,文件信息:" + FileUtil.fileInfo(jarPath));
            try {
                Class<?> aClass = loader.loadClass(cmptDef.getFullQualifiedName());
            } catch (ClassNotFoundException e1) {
                e1.printStackTrace();
                logger.error("类未载入:" + cmptDef.getFullQualifiedName());
            }
        }
        return null;
    }

    /**
     * 执行对象的方法
     *
     * @param o          实例对象
     * @param methodName 方法名称
     * @param args       形参
     * @return
     */
    public static Object invokeMethod(Object o, String methodName, Object... args) {
        try {
            Object invoke;
            if (null == args || args.length == 0) {//有缺陷,如果形参是任意Object,但传入参数是null,无法得到形参类型,得到有参的方法.
                Method method = o.getClass().getMethod(methodName);
                invoke = method.invoke(o);
            } else {
                Method method = o.getClass().getMethod(methodName, args.getClass());
                invoke = method.invoke(o, args);
            }
            return invoke;
        } catch (Exception e) {
            logger.error("",e);
        }
        return null;
    }

}
```

### 组件实现公共接口类

```java
public interface ICmpt {

    /**
     * 组件执行入口
     *
     * @param request
     * @param config，组件实例的参数配置
     * @param actionNode,      当前执行的流程节点
     * @param procContextDTO,     流程引擎实例上下文
     * @return
     */
    CmptResult execute(CmptRequest request, Map<String, FieldDTO> config, ActionNode actionNode, ProcContextDTO procContextDTO);

    /**
     * 销毁组件持有的特殊资源，比如线程。
     */
    void destroy();
}
```

