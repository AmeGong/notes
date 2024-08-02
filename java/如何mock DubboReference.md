# 背景
在集成测试用，@MockBean与@DubboReference不能天然相融，mock的rpc服务不能被自动注入，为了解决这个问题需要进行一些配置。
# Mockito与Dubbo注解分析

为了解决这个问题，我们需要先了解@MockBean与@DubboReference分别在spring容器中做了什么。
Mockito注解在Spring容器中的作用
mockito提供两个注解，分为@Mock与@MockBean，下面分别分析两个注解的作用。
## @Mock注解
该注解需要配置MockitoTestExecutionListener使用，MockitoTestExecutionListener在容器启动之后，测试用例启动之前生效，将测试用例类中被@Mock修饰的属性替换为mockbean，但是该注解并不能替换Spring容器中的对象。下面为了一个伪代码列子：
```
// spring 容器中的类定义
@Service
class A {
  @Autowired
  B b;
}

class Test {

  @Mock
  B b;

  @Autowired
  A a;
  
  @Test
  public test(){
    // 下列对象不是mockbean
    a.b;

    // 下列对象才是mockbean
    b;
  }
}
```

## @MockBean注解
该注解在Spring容器启动时生效，在Spring容器启动时，MockitoPostProcessor创建会将所有被@MockBean修饰的beandefinition传进来，在invokeBeanFactoryPostProcessors方法中，实例化所有@MockBean修饰的实例并注入到Spring容器中，并且MockBean的名字会以#0结尾(MockitoPostProcessor#getBeanName)。详细注入过程见MockitoPostProcessor类。

# Dubbo注解在Spring容器中的作用
注解@DubboReference对应的BeanPostProcessor为ReferenceAnnotationBeanPostProcessor，该类会在Bean的实例的属性填充过程中发挥作用，通过构建rpc代理实例，将该代理实例添加到Spring容器中，同时也将该代理实力填充到被@DubboReference修饰的属性。


## 原因
知道了@MockBean和@DubboReference的作用，我们就很容易发现，@MockBean的rpc服务不能被自动注入的根本原因是两个注解对应的BeanPostProcessor的优先级导致的。
有两个原因影响了Dubbo服务的mock失败：
1. 由于@DubboReference对应的BeanPostProcessor优先级低于@MockBean对应的BeanPostProcessor，所以被mock的dubbo服务会被@DubboReference的BeanPostProcessor覆盖。
2. 在测试类中通过@MockBean修饰的mockbean只能通过MockitoTestExecutionListener被注入到测试类中，不能被注入到非测试类中。这是因为MockitoPostProcessor在使用DefinitionsParser解析时，不能解析测试类，从而没有在MockitoPostProcessor中讲测试类中的mockbean进行属性填充。
```
@Service
public class Client {

    @Autowired
    private Facade facade;

    public invoke() {
      facade.invoke();
    }
}
```

```
@TestExecutionListeners(listeners = { MockitoTestExecutionListener.class})
public class ClientTest  {

    @MockBean
    private Client client;

    @Test
    public void strategyConsult_success() {
        // 直接使用mock的bean
        CollStrategyDTO strategyDTO = client.invoke();
    }
}
```

```
@TestExecutionListeners(listeners = { MockitoTestExecutionListener.class})
public class ClientTest  {

    @MockBean
    private Facade facade;

    @Test
    public void strategyConsult_success() {
        // 还是调用的原本的facade
        CollStrategyDTO strategyDTO = client.invoke(null, null);
    }
}
```

## 解决方案
知道了原因后，方案也很明确了，实现一个新的BeanPostProcessor，重新将mockbean填充到对应的实例中，代码实现如下。

```
/**
 * <p>在集成测试中集成{@link @Mockbean}与{@link @DubboReference}的配置类</>
 *
 * <p>原因</p>
 * <p>{@link @DubboReference}的属性注入在spring容器初始化的Bean填充阶段，即{@link AbstractAutowireCapableBeanFactory#populateBean}方法，该方法会调用BeanPostProcessor进行Bean实例的属性填充。</p>
 * <p>由于{@link @MockBean}对应的BeanPostProcessor的优先级为Ordered.HIGHEST_PRECEDENCE，且该接口没有实现{@link MergedBeanDefinitionPostProcessor}，所以生成的MockBean会在{@link ReferenceAnnotationBeanPostProcessor}被重新覆盖，从而导致MockBean失效
 *
 * <p>解决方案</p>
 * <p>为了解决这个问题，在测试环境实现一个新的BeanPostProcessor，实现{@link PriorityOrdered}和{@link MergedBeanDefinitionPostProcessor}
 * <p>1. 优先级需要低于{@link ReferenceAnnotationBeanPostProcessor}
 * <p>2. 实现{@link MergedBeanDefinitionPostProcessor}的原因:
 * <p>在{@link PostProcessorRegistrationDelegate}的BeanPostProcessor排序中,{@link MergedBeanDefinitionPostProcessor}为最后添加到BeanPostProcessor列表中的{@link ReferenceAnnotationBeanPostProcessor}实现了该接口，会在最后一次添加BeanPostProcessor时重排序是处于一个低优先级，为了保证{@link DubboMockBeanPostProcessor }的优先级始终低于{@link ReferenceAnnotationBeanPostProcessor}，{@link DubboMockBeanPostProcessor }也需要实现{@link MergedBeanDefinitionPostProcessor}</p>
 * @author gongjiawei.gjw
 * @version : DubboMockListener, v0.1 2024年02月04日 13:56 gongjiawei.gjw Exp $
 * @see org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor
 * @see org.springframework.boot.test.mock.mockito.MockitoPostProcessor
 * @see org.springframework.context.support.PostProcessorRegistrationDelegate#registerBeanPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, org.springframework.context.support.AbstractApplicationContext)
 */
public class DubboMockBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter implements MergedBeanDefinitionPostProcessor, BeanFactoryAware, PriorityOrdered {

    private BeanFactory beanFactory;

    private final Class<? extends Annotation>[] annotationTypes;

    /**
     * make sure higher priority than {@link org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor} and {@link org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor}
     */
    private final int order = Ordered.LOWEST_PRECEDENCE - 1;

    public DubboMockBeanPostProcessor() {
        this.annotationTypes = new Class[]{DubboReference .class, Reference .class, com.alibaba.dubbo.config.annotation.Reference.class};

    }

    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {
        Class<?> beanClass = bean.getClass();
        ReflectionUtils.doWithFields(beanClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                for (Class<? extends Annotation> annotationType : getAnnotationTypes()) {
                    Annotation annotation = field.getAnnotation(annotationType);
                    if (annotation != null) {
                        Class<?> declaringClass = field.getType();
                        // @MockBean修饰的bean以"#0"结尾
                        String mockFieldBeanName = declaringClass.getCanonicalName() + "#" + 0;
                        if (beanFactory.containsBean(mockFieldBeanName)) {
                            field.setAccessible(true);
                            // 不能使用getBean(Class<T> requiredType)这个方法，因为@MockBean和@DubboReference会注入两个该类型的bean到spring容器中，导致报错
                            field.set(bean, beanFactory.getBean(mockFieldBeanName));
                            field.setAccessible(false);
                        }
                    }
                }
            }
        });
        return null;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @Override
    public int getOrder() {
        return order;
    }

    protected final Class<? extends Annotation>[] getAnnotationTypes() {
        return annotationTypes;
    }

    /**
     * 必须实现MergedBeanDefinitionPostProcessor,
     * @param beanDefinition
     * @param beanType
     * @param beanName
     * @see org.springframework.context.support.PostProcessorRegistrationDelegate#registerBeanPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, org.springframework.context.support.AbstractApplicationContext)
     */
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        return;
    }
}
```

# 其他
为什么不在初始化后置处理时进行替换？
在spring启动的过程中，如果被aop拦截，则在spring容器中的对象都是代理，在spring的initial阶段就是已经是代理bean了，这个时候比较难对被代理的bean进行属性替换。
为什么不和ReferenceAnnotationBeanPostProcessor一样继承AbstractAnnotationBeanPostProcessor？
在我一实现的思路也是实现AbstractAnnotationBeanPostProcessor，但是发现走不到自己实现的doGetInjectedBean方法。随后debug发现InjectionMetadata#inject中的checkedElements为空，导致没有进入正在的doGetInjectedBean方法。
而checkedElements为空的原因暂时推断为ReferenceAnnotationBeanPostProcessor的AbstractAnnotationBeanPostProcessor#postProcessMergedBeanDefinition方法修改了通过RootBeanDefinition#registerExternallyManagedConfigMember方法修改了BeanDefinition中的属性，导致我们实现的BeanPostProcessor无法再次调用doGetInjectedBean方法。