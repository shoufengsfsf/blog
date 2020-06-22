---
layout: spring源码问题
title: spring源码问题
date: 2020-06-21 22:50:13
tags: spring源码问题
categories: spring源码解读
---

# spring源码问题

## 1、如何统一配置文件的标准？

BeanDefinition

```java
//SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以BeanDefinition来描述的
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

   /**
    * Scope identifier for the standard singleton scope: "singleton".
    * <p>Note that extended bean factories might support further scopes.
    * @see #setScope
    */
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

   /**
    * Scope identifier for the standard prototype scope: "prototype".
    * <p>Note that extended bean factories might support further scopes.
    * @see #setScope
    */
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


   /**
    * Role hint indicating that a {@code BeanDefinition} is a major part
    * of the application. Typically corresponds to a user-defined bean.
    */
   int ROLE_APPLICATION = 0;

   /**
    * Role hint indicating that a {@code BeanDefinition} is a supporting
    * part of some larger configuration, typically an outer
    * {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
    * {@code SUPPORT} beans are considered important enough to be aware
    * of when looking more closely at a particular
    * {@link org.springframework.beans.factory.parsing.ComponentDefinition},
    * but not when looking at the overall configuration of an application.
    */
   int ROLE_SUPPORT = 1;

   /**
    * Role hint indicating that a {@code BeanDefinition} is providing an
    * entirely background role and has no relevance to the end-user. This hint is
    * used when registering beans that are completely part of the internal workings
    * of a {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
    */
   int ROLE_INFRASTRUCTURE = 2;


   // Modifiable attributes

   /**
    * Set the name of the parent definition of this bean definition, if any.
    */
   void setParentName(@Nullable String parentName);

   /**
    * Return the name of the parent definition of this bean definition, if any.
    */
   @Nullable
   String getParentName();

   /**
    * Specify the bean class name of this bean definition.
    * <p>The class name can be modified during bean factory post-processing,
    * typically replacing the original class name with a parsed variant of it.
    * @see #setParentName
    * @see #setFactoryBeanName
    * @see #setFactoryMethodName
    */
   void setBeanClassName(@Nullable String beanClassName);

   /**
    * Return the current bean class name of this bean definition.
    * <p>Note that this does not have to be the actual class name used at runtime, in
    * case of a child definition overriding/inheriting the class name from its parent.
    * Also, this may just be the class that a factory method is called on, or it may
    * even be empty in case of a factory bean reference that a method is called on.
    * Hence, do <i>not</i> consider this to be the definitive bean type at runtime but
    * rather only use it for parsing purposes at the individual bean definition level.
    * @see #getParentName()
    * @see #getFactoryBeanName()
    * @see #getFactoryMethodName()
    */
   @Nullable
   String getBeanClassName();

   /**
    * Override the target scope of this bean, specifying a new scope name.
    * @see #SCOPE_SINGLETON
    * @see #SCOPE_PROTOTYPE
    */
   void setScope(@Nullable String scope);

   /**
    * Return the name of the current target scope for this bean,
    * or {@code null} if not known yet.
    */
   @Nullable
   String getScope();

   /**
    * Set whether this bean should be lazily initialized.
    * <p>If {@code false}, the bean will get instantiated on startup by bean
    * factories that perform eager initialization of singletons.
    */
   void setLazyInit(boolean lazyInit);

   /**
    * Return whether this bean should be lazily initialized, i.e. not
    * eagerly instantiated on startup. Only applicable to a singleton bean.
    */
   boolean isLazyInit();

   /**
    * Set the names of the beans that this bean depends on being initialized.
    * The bean factory will guarantee that these beans get initialized first.
    */
   void setDependsOn(@Nullable String... dependsOn);

   /**
    * Return the bean names that this bean depends on.
    */
   @Nullable
   String[] getDependsOn();

   /**
    * Set whether this bean is a candidate for getting autowired into some other bean.
    * <p>Note that this flag is designed to only affect type-based autowiring.
    * It does not affect explicit references by name, which will get resolved even
    * if the specified bean is not marked as an autowire candidate. As a consequence,
    * autowiring by name will nevertheless inject a bean if the name matches.
    */
   void setAutowireCandidate(boolean autowireCandidate);

   /**
    * Return whether this bean is a candidate for getting autowired into some other bean.
    */
   boolean isAutowireCandidate();

   /**
    * Set whether this bean is a primary autowire candidate.
    * <p>If this value is {@code true} for exactly one bean among multiple
    * matching candidates, it will serve as a tie-breaker.
    */
   void setPrimary(boolean primary);

   /**
    * Return whether this bean is a primary autowire candidate.
    */
   boolean isPrimary();

   /**
    * Specify the factory bean to use, if any.
    * This the name of the bean to call the specified factory method on.
    * @see #setFactoryMethodName
    */
   void setFactoryBeanName(@Nullable String factoryBeanName);

   /**
    * Return the factory bean name, if any.
    */
   @Nullable
   String getFactoryBeanName();

   /**
    * Specify a factory method, if any. This method will be invoked with
    * constructor arguments, or with no arguments if none are specified.
    * The method will be invoked on the specified factory bean, if any,
    * or otherwise as a static method on the local bean class.
    * @see #setFactoryBeanName
    * @see #setBeanClassName
    */
   void setFactoryMethodName(@Nullable String factoryMethodName);

   /**
    * Return a factory method, if any.
    */
   @Nullable
   String getFactoryMethodName();

   /**
    * Return the constructor argument values for this bean.
    * <p>The returned instance can be modified during bean factory post-processing.
    * @return the ConstructorArgumentValues object (never {@code null})
    */
   ConstructorArgumentValues getConstructorArgumentValues();

   /**
    * Return if there are constructor argument values defined for this bean.
    * @since 5.0.2
    */
   default boolean hasConstructorArgumentValues() {
      return !getConstructorArgumentValues().isEmpty();
   }

   /**
    * Return the property values to be applied to a new instance of the bean.
    * <p>The returned instance can be modified during bean factory post-processing.
    * @return the MutablePropertyValues object (never {@code null})
    */
   MutablePropertyValues getPropertyValues();

   /**
    * Return if there are property values values defined for this bean.
    * @since 5.0.2
    */
   default boolean hasPropertyValues() {
      return !getPropertyValues().isEmpty();
   }


   // Read-only attributes

   /**
    * Return whether this a <b>Singleton</b>, with a single, shared instance
    * returned on all calls.
    * @see #SCOPE_SINGLETON
    */
   boolean isSingleton();

   /**
    * Return whether this a <b>Prototype</b>, with an independent instance
    * returned for each call.
    * @since 3.0
    * @see #SCOPE_PROTOTYPE
    */
   boolean isPrototype();

   /**
    * Return whether this bean is "abstract", that is, not meant to be instantiated.
    */
   boolean isAbstract();

   /**
    * Get the role hint for this {@code BeanDefinition}. The role hint
    * provides the frameworks as well as tools with an indication of
    * the role and importance of a particular {@code BeanDefinition}.
    * @see #ROLE_APPLICATION
    * @see #ROLE_SUPPORT
    * @see #ROLE_INFRASTRUCTURE
    */
   int getRole();

   /**
    * Return a human-readable description of this bean definition.
    */
   @Nullable
   String getDescription();

   /**
    * Return a description of the resource that this bean definition
    * came from (for the purpose of showing context in case of errors).
    */
   @Nullable
   String getResourceDescription();

   /**
    * Return the originating BeanDefinition, or {@code null} if none.
    * Allows for retrieving the decorated bean definition, if any.
    * <p>Note that this method returns the immediate originator. Iterate through the
    * originator chain to find the original BeanDefinition as defined by the user.
    */
   @Nullable
   BeanDefinition getOriginatingBeanDefinition();

}
```

## 2、IOC容器最顶层接口

BeanFactory

```java
//在BeanFactory里只对IOC容器的基本行为作了定义，根本不关心你的Bean是如何定义怎样加载的。
//正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。
public interface BeanFactory {

   /**
    * Used to dereference a {@link FactoryBean} instance and distinguish it from
    * beans <i>created</i> by the FactoryBean. For example, if the bean named
    * {@code myJndiObject} is a FactoryBean, getting {@code &myJndiObject}
    * will return the factory, not the instance returned by the factory.
    */
   //对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，
   //如果需要得到工厂本身，需要转义
   String FACTORY_BEAN_PREFIX = "&";


   /**
    * Return an instance, which may be shared or independent, of the specified bean.
    * <p>This method allows a Spring BeanFactory to be used as a replacement for the
    * Singleton or Prototype design pattern. Callers may retain references to
    * returned objects in the case of Singleton beans.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to retrieve
    * @return an instance of the bean
    * @throws NoSuchBeanDefinitionException if there is no bean definition
    * with the specified name
    * @throws BeansException if the bean could not be obtained
    */
   //根据bean的名字，获取在IOC容器中得到bean实例
   Object getBean(String name) throws BeansException;

   /**
    * Return an instance, which may be shared or independent, of the specified bean.
    * <p>Behaves the same as {@link #getBean(String)}, but provides a measure of type
    * safety by throwing a BeanNotOfRequiredTypeException if the bean is not of the
    * required type. This means that ClassCastException can't be thrown on casting
    * the result correctly, as can happen with {@link #getBean(String)}.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to retrieve
    * @param requiredType type the bean must match. Can be an interface or superclass
    * of the actual class, or {@code null} for any match. For example, if the value
    * is {@code Object.class}, this method will succeed whatever the class of the
    * returned instance.
    * @return an instance of the bean
    * @throws NoSuchBeanDefinitionException if there is no such bean definition
    * @throws BeanNotOfRequiredTypeException if the bean is not of the required type
    * @throws BeansException if the bean could not be created
    */
   //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。
   <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;

   /**
    * Return an instance, which may be shared or independent, of the specified bean.
    * <p>Allows for specifying explicit constructor arguments / factory method arguments,
    * overriding the specified default arguments (if any) in the bean definition.
    * @param name the name of the bean to retrieve
    * @param args arguments to use when creating a bean instance using explicit arguments
    * (only applied when creating a new instance as opposed to retrieving an existing one)
    * @return an instance of the bean
    * @throws NoSuchBeanDefinitionException if there is no such bean definition
    * @throws BeanDefinitionStoreException if arguments have been given but
    * the affected bean isn't a prototype
    * @throws BeansException if the bean could not be created
    * @since 2.5
    */
   Object getBean(String name, Object... args) throws BeansException;

   /**
    * Return the bean instance that uniquely matches the given object type, if any.
    * <p>This method goes into {@link ListableBeanFactory} by-type lookup territory
    * but may also be translated into a conventional by-name lookup based on the name
    * of the given type. For more extensive retrieval operations across sets of beans,
    * use {@link ListableBeanFactory} and/or {@link BeanFactoryUtils}.
    * @param requiredType type the bean must match; can be an interface or superclass.
    * {@code null} is disallowed.
    * @return an instance of the single bean matching the required type
    * @throws NoSuchBeanDefinitionException if no bean of the given type was found
    * @throws NoUniqueBeanDefinitionException if more than one bean of the given type was found
    * @throws BeansException if the bean could not be created
    * @since 3.0
    * @see ListableBeanFactory
    */
   <T> T getBean(Class<T> requiredType) throws BeansException;

   /**
    * Return an instance, which may be shared or independent, of the specified bean.
    * <p>Allows for specifying explicit constructor arguments / factory method arguments,
    * overriding the specified default arguments (if any) in the bean definition.
    * <p>This method goes into {@link ListableBeanFactory} by-type lookup territory
    * but may also be translated into a conventional by-name lookup based on the name
    * of the given type. For more extensive retrieval operations across sets of beans,
    * use {@link ListableBeanFactory} and/or {@link BeanFactoryUtils}.
    * @param requiredType type the bean must match; can be an interface or superclass.
    * {@code null} is disallowed.
    * @param args arguments to use when creating a bean instance using explicit arguments
    * (only applied when creating a new instance as opposed to retrieving an existing one)
    * @return an instance of the bean
    * @throws NoSuchBeanDefinitionException if there is no such bean definition
    * @throws BeanDefinitionStoreException if arguments have been given but
    * the affected bean isn't a prototype
    * @throws BeansException if the bean could not be created
    * @since 4.1
    */
   <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;


   /**
    * Does this bean factory contain a bean definition or externally registered singleton
    * instance with the given name?
    * <p>If the given name is an alias, it will be translated back to the corresponding
    * canonical bean name.
    * <p>If this factory is hierarchical, will ask any parent factory if the bean cannot
    * be found in this factory instance.
    * <p>If a bean definition or singleton instance matching the given name is found,
    * this method will return {@code true} whether the named bean definition is concrete
    * or abstract, lazy or eager, in scope or not. Therefore, note that a {@code true}
    * return value from this method does not necessarily indicate that {@link #getBean}
    * will be able to obtain an instance for the same name.
    * @param name the name of the bean to query
    * @return whether a bean with the given name is present
    */
   //提供对bean的检索，看看是否在IOC容器有这个名字的bean
   boolean containsBean(String name);

   /**
    * Is this bean a shared singleton? That is, will {@link #getBean} always
    * return the same instance?
    * <p>Note: This method returning {@code false} does not clearly indicate
    * independent instances. It indicates non-singleton instances, which may correspond
    * to a scoped bean as well. Use the {@link #isPrototype} operation to explicitly
    * check for independent instances.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to query
    * @return whether this bean corresponds to a singleton instance
    * @throws NoSuchBeanDefinitionException if there is no bean with the given name
    * @see #getBean
    * @see #isPrototype
    */
   //根据bean名字得到bean实例，并同时判断这个bean是不是单例
   boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

   /**
    * Is this bean a prototype? That is, will {@link #getBean} always return
    * independent instances?
    * <p>Note: This method returning {@code false} does not clearly indicate
    * a singleton object. It indicates non-independent instances, which may correspond
    * to a scoped bean as well. Use the {@link #isSingleton} operation to explicitly
    * check for a shared singleton instance.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to query
    * @return whether this bean will always deliver independent instances
    * @throws NoSuchBeanDefinitionException if there is no bean with the given name
    * @since 2.0.3
    * @see #getBean
    * @see #isSingleton
    */
   boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

   /**
    * Check whether the bean with the given name matches the specified type.
    * More specifically, check whether a {@link #getBean} call for the given name
    * would return an object that is assignable to the specified target type.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to query
    * @param typeToMatch the type to match against (as a {@code ResolvableType})
    * @return {@code true} if the bean type matches,
    * {@code false} if it doesn't match or cannot be determined yet
    * @throws NoSuchBeanDefinitionException if there is no bean with the given name
    * @since 4.2
    * @see #getBean
    * @see #getType
    */
   boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

   /**
    * Check whether the bean with the given name matches the specified type.
    * More specifically, check whether a {@link #getBean} call for the given name
    * would return an object that is assignable to the specified target type.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to query
    * @param typeToMatch the type to match against (as a {@code Class})
    * @return {@code true} if the bean type matches,
    * {@code false} if it doesn't match or cannot be determined yet
    * @throws NoSuchBeanDefinitionException if there is no bean with the given name
    * @since 2.0.1
    * @see #getBean
    * @see #getType
    */
   boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

   /**
    * Determine the type of the bean with the given name. More specifically,
    * determine the type of object that {@link #getBean} would return for the given name.
    * <p>For a {@link FactoryBean}, return the type of object that the FactoryBean creates,
    * as exposed by {@link FactoryBean#getObjectType()}.
    * <p>Translates aliases back to the corresponding canonical bean name.
    * Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the name of the bean to query
    * @return the type of the bean, or {@code null} if not determinable
    * @throws NoSuchBeanDefinitionException if there is no bean with the given name
    * @since 1.1.2
    * @see #getBean
    * @see #isTypeMatch
    */
   //得到bean实例的Class类型
   @Nullable
   Class<?> getType(String name) throws NoSuchBeanDefinitionException;

   /**
    * Return the aliases for the given bean name, if any.
    * All of those aliases point to the same bean when used in a {@link #getBean} call.
    * <p>If the given name is an alias, the corresponding original bean name
    * and other aliases (if any) will be returned, with the original bean name
    * being the first element in the array.
    * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
    * @param name the bean name to check for aliases
    * @return the aliases, or an empty array if none
    * @see #getBean
    */
   //得到bean的别名，如果根据别名检索，那么其原名也会被检索出来
   String[] getAliases(String name);

}
```

