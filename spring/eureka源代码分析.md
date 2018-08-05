# <u>Eureka client源码分析一</u>

没怎么玩spring，结果还得回头看看spring这些注解的初始化机制，下面@Inherited表示标记类的子类可以继承该注解；@Import里面有个EnableDiscoveryClientImportSelector类，这个类表示如何加载.

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

   /**
    * If true, the ServiceRegistry will automatically register the local server.
    */
   boolean autoRegister() default true;
}
```

下面看下EnableDiscoveryClientImportSelector.class源码如下：

```
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
      extends SpringFactoryImportSelector<EnableDiscoveryClient> {

   @Override
   public String[] selectImports(AnnotationMetadata metadata) {
      String[] imports = super.selectImports(metadata);

      AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

      boolean autoRegister = attributes.getBoolean("autoRegister");

      if (autoRegister) {
         List<String> importsList = new ArrayList<>(Arrays.asList(imports));
         //加载AutoServiceRegistrationConfiguration.class
              importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
         imports = importsList.toArray(new String[0]);
      } else {
         Environment env = getEnvironment();
         if(ConfigurableEnvironment.class.isInstance(env)) {
            ConfigurableEnvironment configEnv = (ConfigurableEnvironment)env;
            LinkedHashMap<String, Object> map = new LinkedHashMap<>();
            map.put("spring.cloud.service-registry.auto-registration.enabled", false);
            MapPropertySource propertySource = new MapPropertySource(
                  "springCloudDiscoveryClient", map);
            configEnv.getPropertySources().addLast(propertySource);
         }

      }

      return imports;
   }

   @Override
   protected boolean isEnabled() {
      return new RelaxedPropertyResolver(getEnvironment()).getProperty(
            "spring.cloud.discovery.enabled", Boolean.class, Boolean.TRUE);
   }

   @Override
   protected boolean hasDefaultFactory() {
      return true;
   }
```
其实就是加载配置AutoServiceRegistrationConfiguration

spring boot  @EnableAutoConfiguration作用 会加载spring-cloud-netflix-eureka-client下的spring.factoreis文件:

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
```

