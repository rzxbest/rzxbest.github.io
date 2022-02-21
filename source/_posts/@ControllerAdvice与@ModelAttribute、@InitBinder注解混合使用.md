---
title: ExceptionHandler和ControllerAdvice注解分析
date: 2022-02-21 19:59:29
tags: [JAVA,Spring]
categories: [JAVA,Spring]
top: 20
---

# @ControllerAdvice与@ModelAttribute、@InitBinder注解混合使用

## @ModelAttribute注解的作用

- 用在方法的参数上
	- 注解在参数上，会将客户端传递过来的参数按名称注入到指定对象中，
	- 并且会将这个对象自动加入ModelMap中
- 用在Controller的方法上
	- 注解在方法上，会在每一个@RequestMapping标注的方法前执行，
	- 如果有返回值，则自动将该返回值加入到ModelMap中
	
## @InitBinder注解混合使用

- 对数据绑定进行设置
	WebDataBinder中有很多方法可以对数据绑定进行具体的设置
```
@InitBinder
public void initBinder(WebDataBinder binder) {
	binder.setDisallowedFields("name");
}
```


- 注册已有封装好的编辑器
	- 由于前台传到controller里的值是String类型的，当往Model里Set这个值的时候，如果set的这个属性是个对象，Spring就会去找到对应的editor进行转换，然后再set进去
	- Spring自己提供了大量的实现类（如下图所示的在org.springframwork.beans.propertyEditors下的所有editor），诸如CustomDateEditor ，CustomBooleanEditor，CustomNumberEditor等许多，基本上够用

## 如何加载	RequestMappingHandlerAdapter？

自己注入或者使用默认的
```
private void initHandlerAdapters(ApplicationContext context) {
	this.handlerAdapters = null;

	if (this.detectAllHandlerAdapters) { //true
		// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerAdapter> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);//从容器中获取HandlerAdapter.class
		if (!matchingBeans.isEmpty()) {
			this.handlerAdapters = new ArrayList<>(matchingBeans.values());
			// We keep HandlerAdapters in sorted order.
			AnnotationAwareOrderComparator.sort(this.handlerAdapters);
		}
	}
	else {
		try {
			HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
			this.handlerAdapters = Collections.singletonList(ha);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default HandlerAdapter later.
		}
	}

	// Ensure we have at least some HandlerAdapters, by registering
	// default HandlerAdapters if no other adapters are found.
	if (this.handlerAdapters == null) {
		this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);//从DispatchServlet.properties中获取
		if (logger.isTraceEnabled()) {
			logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
					"': using default strategies from DispatcherServlet.properties");
		}
	}
}
	
```

## DispatchServlet.properties中获取HandlerAdapter

```
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter
```

## 何时加载@ControllerAdvice与@ModelAttribute、@InitBinder注解？

RequestMappingHandlerAdapter类实现了InitializingBean接口，在afterPropertiesSet方法中initControllerAdviceCache方法中加载

```
public static final MethodFilter INIT_BINDER_METHODS = method ->
			AnnotatedElementUtils.hasAnnotation(method, InitBinder.class);


public static final MethodFilter MODEL_ATTRIBUTE_METHODS = method ->
		(!AnnotatedElementUtils.hasAnnotation(method, RequestMapping.class) &&
				AnnotatedElementUtils.hasAnnotation(method, ModelAttribute.class));

private void initControllerAdviceCache() {
	if (getApplicationContext() == null) {
		return;
	}

	//从容器中获取ControllerAdvice注解的bean
	List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());

	List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();
	//遍历每个bean,找出InitBinder注解的方法放入initBinderAdviceCache里，找出ModelAttribute注解的方法放入modelAttributeAdviceCache
	for (ControllerAdviceBean adviceBean : adviceBeans) {
		Class<?> beanType = adviceBean.getBeanType();
		if (beanType == null) {
			throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
		}
		Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
		if (!attrMethods.isEmpty()) {
			this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
		}
		Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
		if (!binderMethods.isEmpty()) {
			this.initBinderAdviceCache.put(adviceBean, binderMethods);
		}
		if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
			requestResponseBodyAdviceBeans.add(adviceBean);
		}
	}

	...省略
}

```

## 何时调用 @ModelAttribute、@InitBinder注解的方法或参数？

- 在RequestMappingHandlerAdapter对象调用handle方法中调用
- handle方法调用handleInternal方法,handleInternal方法调用invokeHandlerMethod方法
- invokeHandlerMethod 方法中getDataBinderFactory方法获取InitBinder注解的方法
- invokeHandlerMethod 方法中getModelFactory方法获取ModelAttribute注解的方法
- 调用ModelFactory对象的initModel方法，initModel方法中调用invokeModelAttributeMethods方法，调用@ModelAttribute注解的方法
- 调用ModelFactory对象的initModel方法，initModel方法中调用findSessionAttributeArguments方法，对@ModelAttribute注解的参数赋值
-- invocableMethod.invokeAndHandle(webRequest, mavContainer)，在这个方法中调用invokeForRequest(),invokeForRequest()中调用getMethodArgumentValues()方法对参数进行了绑定

## getDataBinderFactory方法 getModelFactory方法

- 先从controller类获取@InitBinder注解的方法
- 在从@ControllerAdvice注解的bean上获取@InitBinder注解的方法
- 上述的方法组合并返回ModelFactory

```
private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
	Class<?> handlerType = handlerMethod.getBeanType();
	Set<Method> methods = this.initBinderCache.get(handlerType);
	if (methods == null) {
		methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
		this.initBinderCache.put(handlerType, methods);
	}
	List<InvocableHandlerMethod> initBinderMethods = new ArrayList<>();
	// Global methods first
	this.initBinderAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
		if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
			Object bean = controllerAdviceBean.resolveBean();
			for (Method method : methodSet) {
				initBinderMethods.add(createInitBinderMethod(bean, method));
			}
		}
	});
	for (Method method : methods) {
		Object bean = handlerMethod.getBean();
		initBinderMethods.add(createInitBinderMethod(bean, method));
	}
	return createDataBinderFactory(initBinderMethods);
}
```

- 先从controller类获取@ModelAttribute注解的方法
- 在从@ControllerAdvice注解的bean上获取@ModelAttribute注解的方法
- 上述的方法组合并返回ModelFactory
```
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
	SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
	Class<?> handlerType = handlerMethod.getBeanType();
	Set<Method> methods = this.modelAttributeCache.get(handlerType);
	if (methods == null) {
		methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
		this.modelAttributeCache.put(handlerType, methods);
	}
	List<InvocableHandlerMethod> attrMethods = new ArrayList<>();
	// Global methods first
	this.modelAttributeAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
		if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
			Object bean = controllerAdviceBean.resolveBean();
			for (Method method : methodSet) {
				attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
			}
		}
	});
	for (Method method : methods) {
		Object bean = handlerMethod.getBean();
		attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
	}
	return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}

```

## invokeModelAttributeMethods方法

```

private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
			throws Exception {

	while (!this.modelMethods.isEmpty()) {
		InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
		ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
		Assert.state(ann != null, "No ModelAttribute annotation");
		if (container.containsAttribute(ann.name())) {
			if (!ann.binding()) {
				container.setBindingDisabled(ann.name());
			}
			continue;
		}

		Object returnValue = modelMethod.invokeForRequest(request, container);
		if (modelMethod.isVoid()) {
			if (StringUtils.hasText(ann.value())) {
				if (logger.isDebugEnabled()) {
					logger.debug("Name in @ModelAttribute is ignored because method returns void: " +
							modelMethod.getShortLogMessage());
				}
			}
			continue;
		}

		String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
		if (!ann.binding()) {
			container.setBindingDisabled(returnValueName);
		}
		if (!container.containsAttribute(returnValueName)) {
			container.addAttribute(returnValueName, returnValue);
		}
	}
}
```

## findSessionAttributeArguments方法

```
private List<String> findSessionAttributeArguments(HandlerMethod handlerMethod) {
	List<String> result = new ArrayList<>();
	for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
		if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
			String name = getNameForParameter(parameter);
			Class<?> paramType = parameter.getParameterType();
			if (this.sessionAttributesHandler.isHandlerSessionAttribute(name, paramType)) {
				result.add(name);
			}
		}
	}
	return result;
}
```

## getMethodArgumentValues方法

```

protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	if (ObjectUtils.isEmpty(parameters)) {
		return EMPTY_ARGS;
	}

	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		args[i] = findProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (!this.resolvers.supportsParameter(parameter)) {
			throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
		}
		try {
			args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
		}
		catch (Exception ex) {
			// Leave stack trace for later, exception may actually be resolved and handled...
			if (logger.isDebugEnabled()) {
				String exMsg = ex.getMessage();
				if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
					logger.debug(formatArgumentError(parameter, exMsg));
				}
			}
			throw ex;
		}
	}
	return args;
}

```