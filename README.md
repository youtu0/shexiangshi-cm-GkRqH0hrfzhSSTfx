
最近把Spring Boot的版本升级到了3\.3\.5，突然发现一个问题：当使用Spring Data JPA自动生成表的时候，所产生的列顺序与Entity类中的变量顺序不一致了。比如，有一个下面这样的Entity：



```
@Data
@Entity(name = "t_config")
@EntityListeners(AuditingEntityListener.class)
public class Config {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(length = 20)
    private String itemKey;
    @Column(length = 200)
    private String itemValue;
    @Column(length = 200)
    private String itemDesc;

    @CreatedDate
    private Date createTime;
    @LastModifiedDate
    private Date modifyTime;

}

```

实际自动创建出来的是这样的：


![](https://img2024.cnblogs.com/blog/626506/202411/626506-20241127123732092-621778261.png)


自动创建的表结构中各个列与Entity类中的变量顺序不一致。其实该问题是一个老生常谈的问题了，在[DD](https://github.com)这次升级的工程里是有做过解决方案的。只是升级了Spring Boot版本之后，之前的解决方案失效了。


搜索了一番，同时还问了一下AI，发现给出的方案还都是老的解决方案，所以今天特别写一篇来记录下新版本之下，要如何解决这个问题。如果您刚好遇到类似的问题，可以参考本文来解决。


## 老版本解决方案


新老版本的解决思路是类似的，都是替换Hibernate的实现，下面是老版本的解决步骤：


1. 在工程中新建`org.hibernate.cfg`包
2. 找到`hibernate-core`包下的`org.hibernate.cfg`下的`PropertyContainer`类，复制到本工程的`org.hibernate.cfg`包下
3. 把`PropertyContainer`类中定义的`persistentAttributeMap`类型从`TreeMap`修改为`LinkedHashMap`


## 新版本解决方案


虽然之前的方案失效了，但思路应该还是对的，所以第一反应是看看当前版本下的`PropertyContainer`类，具体如下（省略了一些不重要的内容）：



```
package org.hibernate.boot.model.internal;

//省略...

public class PropertyContainer {

    private static final CoreMessageLogger LOG = Logger.getMessageLogger(CoreMessageLogger.class, PropertyContainer.class.getName());

    /**
     * The class for which this container is created.
     */
    private final XClass xClass;
    private final XClass entityAtStake;

    /**
     * Holds the AccessType indicated for use at the class/container-level for cases where persistent attribute
     * did not specify.
     */
    private final AccessType classLevelAccessType;

    private final List persistentAttributes;

	//省略...

}

```

可以看到有两个重要变化部分：


1. `PropertyContainer`类的包名从`org.hibernate.cfg`改到了`org.hibernate.boot.model.internal`
2. 之前的`TreeMap persistentAttributeMap`变量没有了，但多了一个`List persistentAttributes`。进一步观察这个新变量的处理过程，可以看到如下逻辑：


![](https://img2024.cnblogs.com/blog/626506/202411/626506-20241127123743111-684706269.png)


所以，新版的方案就以下两个步骤：


1. 在工程中新建`org.hibernate.boot.model.internal`包
2. 找到`hibernate-core`包下的`org.hibernate.boot.model.internal`下的`PropertyContainer`类，复制到本工程的`org.hibernate.boot.model.internal`包下
3. 把`PropertyContainer`类中，上面图中红色圈出部门定义的`localAttributeMap = new TreeMap<>();`修改为`localAttributeMap = new LinkedHashMap<>();`


到这里，在新版本中的这个问题就解决了。如果你也遇到了类似的问题，希望本文对你有所帮助。另外，欢迎加入我们的[Spring技术交流群](https://github.com):[悠兔机场](https://xinnongbo.com)，参与交流与讨论，更好的学习与进步！



> 欢迎关注我的公众号：程序猿DD。第一时间了解前沿行业消息、分享深度技术干货、获取优质学习资源


