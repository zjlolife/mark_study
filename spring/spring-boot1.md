# spring boot加载机制

入口代码如下：下面Application入口类是，main方法运行后仅仅会扫描当前同级包以及子包下的注解。

```
@SpringBootApplication
public class Application {

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

下面的CompoentScan默认会扫描SpringBootApplication下的包的注解，并filter排除掉一些不符合条件的。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
      @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

@EnableAutoConfiguration 这个比较关键,EnableAutoConfigurationImportSelector这个DeferredImportSelector会先获取spring.factories下key为EnableAutoConfiguration的所有注解。并过滤掉不满足OnClassCondition的注解。

```
SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

EnableAutoConfigurationImportSelector类继承AutoConfigurationImportSelector：过滤流程如上面所说

```
public class AutoConfigurationImportSelector
      implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
      BeanFactoryAware, EnvironmentAware, Ordered {

   private static final String[] NO_IMPORTS = {};

   private static final Log logger = LogFactory
         .getLog(AutoConfigurationImportSelector.class);

   private ConfigurableListableBeanFactory beanFactory;

   private Environment environment;

   private ClassLoader beanClassLoader;

   private ResourceLoader resourceLoader;

   @Override
   public String[] selectImports(AnnotationMetadata annotationMetadata) {
      if (!isEnabled(annotationMetadata)) {
         return NO_IMPORTS;
      }
      try {
         AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
               .loadMetadata(this.beanClassLoader);
         AnnotationAttributes attributes = getAttributes(annotationMetadata);
         List<String> configurations = getCandidateConfigurations(annotationMetadata,
               attributes);
         configurations = removeDuplicates(configurations);
         configurations = sort(configurations, autoConfigurationMetadata);
         Set<String> exclusions = getExclusions(annotationMetadata, attributes);
         checkExcludedClasses(configurations, exclusions);
         configurations.removeAll(exclusions);
         configurations = filter(configurations, autoConfigurationMetadata);
         fireAutoConfigurationImportEvents(configurations, exclusions);
         return configurations.toArray(new String[configurations.size()]);
      }
      catch (IOException ex) {
         throw new IllegalStateException(ex);
      }
   }
```

过滤后的Configuration注解 会进行初始化，至此spring boot 初始化流程完成。



##### 参考链接：

[1]: https://www.cnblogs.com/jiaoqq/p/7678037.html
[2]: https://blog.csdn.net/kmhysoft/article/details/71056027
[3]: https://blog.csdn.net/syy_c_j/article/details/52058035

