- [ ] Dependency Injection & Inversion of Control
- [ ] Bean lifecycle and scopes (singleton, prototype, etc.)
- [ ] @Component, @Service, @Repository distinctions
- [ ] ApplicationContext vs BeanFactory
- [ ] Aspect-Oriented Programming concepts (Advice, Pointcut, Aspect)

## 1. **Dependency Injection & Inversion of Control (IoC)**

1. ***What is Inversion of Control (IoC) and how is it implemented in Spring?***
	- IoC = DI (a process)
	- objects define their dependencies (i.e. the other objects they work with). how?
	- only through 3 ways :
		- **Constructor** arguments
		- arguments to a **(static) factory method**, or
		- **properties** that are set on the object instance after it is constructed or returned from a factory method
	- For each bean, its dependencies are expressed in the form of properties, constructor arguments, or arguments to the static-factory method
	- packages that form basis for spring's IoC container : 
		- `org.springframework.beans` 
		- `org.springframework.context`

| BeanFactory                                                                                          | ApplicationContext                          |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| The interface provides an advanced configuration<br>mechanism capable of managing any type of object | a sub-interface of BeanFactory              |
| provides the configuration framework and basic functionality                                         | adds more enterprise-specific functionality |
|                                                                                                      | a complete superset of the BeanFactory      |
Beans =  an object that is instantiated, assembled, and managed by a Spring IoC container.
- A bean definition is essentially a recipe for creating one or more objects.
![[Spring container.png]]
Spring Container
- Configuration Metadata
- Instantiating a Container
- Using the Container

- Spring IoC container = `org.springframework.context.ApplicationContext` interface
	- manages (**i**nstantiates, **c**onfigures, and **a**ssembes) the ***beans***. 
	-  **configuration metadata**. defines instructions on what objects to manage(i,c,a) in 3 possible ways : 
		- XML, (`web.xml`)
		- Java annotations, (`@Component`) or 
		- Java code (`@Configuration`, `@Bean`, `@Import`, and `@DependsOn`): define beans external to your application classes by using Java rather than XML files. 

| Metadata forms                 |                               | Instantiating a Container                                                                      |
| ------------------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------- |
| XML-based configuration        | `<bean id="..." class="...">` | `ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");` |
| Annotation-based configuration |                               |                                                                                                |
| Java-based configuration       |                               |                                                                                                |



1. ***What is Dependency Injection? Explain constructor vs setter injection in Spring.***
	1. 
2. Describe the benefits of using Dependency Injection in application development.
	1. 
3. How does Spring manage dependencies using annotations vs XML configuration?
	1. 
4. What is the difference between tight coupling and loose coupling? How does Spring achieve loose coupling?
	1. 
5. How does the `@Autowired` annotation work? Can it be used with both constructors and methods?
	1. 
6. What problems might occur with Dependency Injection and how would you troubleshoot them?
	1. 
## 2. **Bean Lifecycle and Scopes**

1. What are the main phases of the Spring bean lifecycle?
2. How can you customize bean initialization and destruction?
3. What are the different bean scopes in Spring? When would you use each?
4. How does singleton scope behave compared to prototype scope?
5. How do `@PostConstruct` and `@PreDestroy` annotations work in the lifecycle?
6. What is the role of `BeanPostProcessor` in bean lifecycle management?
7. How would you manage resources or connections during bean destruction?

## 3. **@Component, @Service, @Repository Distinctions**

1. What is the purpose of the `@Component` annotation?
2. How does `@Service` differ from `@Component`? Give a use-case example.
3. What special behavior does `@Repository` provide?
4. Why use specialized stereotypes instead of just `@Component` in large projects?
5. What architectural layer is typically marked with each stereotype annotation?
6. How do these annotations affect exception handling or transaction management?
7. Can you combine multiple stereotype annotations on one class? What happens?

## 4. **ApplicationContext vs BeanFactory**

1. What is the difference between `BeanFactory` and `ApplicationContext`?
2. Explain lazy vs eager bean instantiation in these containers.
3. What advanced features does `ApplicationContext` provide that `BeanFactory` does not?
4. How do you create and configure an `ApplicationContext`?
5. In which scenarios would you prefer using `BeanFactory`?
6. How does event publishing work in `ApplicationContext`?
7. What is dependency resolution and how do these containers perform it?

## 5. **Aspect-Oriented Programming (AOP) Concepts**

1. What is Aspect-Oriented Programming and how is it applied in Spring?
2. Define and distinguish Advice, Pointcut, and Aspect in Spring AOP.
3. What are cross-cutting concerns, and why do they need AOP?
4. Give examples of situations where you would use AOP in a Spring project.
5. What types of Advice exist in Spring AOP?
6. How do you declare an Aspect using annotations?
7. What are the limitations of Spring AOP compared to full AspectJ?



