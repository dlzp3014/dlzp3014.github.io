---
layout: post
title:  "Spring源码-MultipartHttpServletRequest多部分httpServlet请求"
date:   2019-08-09 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Web
---

* content
{:toc}

MultipartHttpServletRequest接口提供了附加的方法用于处理Servlet请求中的multipart内容，允许访问上下文件，接口实现这也需要重写标准的ServletRequest方法用于访问参数，使multipart参数可用Spring中提供的实现有DefaultMultipartHttpServletRequest和StandardMultipartHttpServletRequest，分别对应CommonsMultipartResolver和StandardServletMultipartResolver类，具体参看：[Spring源码-MultipartResolver MultipartRequest解析器](/2019/08/08/spring-MultipartResolver/)

- MultipartHttpServletRequest接口分别继承了HttpServletRequest和MultipartRequest接口，其中MultipartRequest接口定义了multipart请求访问操作，用于暴露真实的multipart请求

- MultipartHttpServletRequest接口继承关系如下：



## MultipartRequest接口定义

```java
/**
 * This interface defines the multipart request access operations that are exposed
 * for actual multipart requests. It is extended 扩展by {@link MultipartHttpServletRequest}.
 *
 * @author Juergen Hoeller
 * @author Arjen Poutsma
 * @since 2.5.2
 */
public interface MultipartRequest {

	/**
	 * Return an {@link java.util.Iterator} of String objects containing the
	 * parameter names of the multipart files contained in this request. These
	 * are the field names属性名 of the form (like with normal parameters), not the
	 * original file names.不是原始文件名
	 * @return the names of the files
	 */
	Iterator<String> getFileNames(); //返回当前请求中包含的multipart文件参数名的迭代对象

	/**
	 * Return the contents plus description of an uploaded file in this request, 
	 * or {@code null} if it does not exist. 不存在时返回null
	 * @param name a String specifying the parameter name of the multipart file
	 * @return the uploaded content in the form of a {@link MultipartFile} object
	 */
	@Nullable
	MultipartFile getFile(String name); //返回上传文件

	/**
	 * Return the contents plus description of uploaded files in this request,
	 * or an empty list if it does not exist.
	 * @param name a String specifying the parameter name of the multipart file
	 * @return the uploaded content in the form of a {@link MultipartFile} list
	 * @since 3.0
	 */
	List<MultipartFile> getFiles(String name); //列表

	/**
	 * Return a {@link java.util.Map} of the multipart files contained in this request.
	 * @return a map containing the parameter names as keys, and the
	 * {@link MultipartFile} objects as values
	 */
	Map<String, MultipartFile> getFileMap(); //映射。一个文件属性名对应一个文件 

	/**
	 * Return a {@link MultiValueMap} of the multipart files contained in this request.
	 * @return a map containing the parameter names as keys, and a list of
	 * {@link MultipartFile} objects as values
	 * @since 3.0
	 */
	MultiValueMap<String, MultipartFile> getMultiFileMap();//多值映射，一个文件属性名对应多个文件

	/**
	 * Determine the content type of the specified request part.
	 * @param paramOrFileName the name of the part
	 * @return the associated content type, or {@code null} if not defined
	 * @since 3.1
	 */
	@Nullable
	String getMultipartContentType(String paramOrFileName); //确定指定请求部分的内容类型

}
```

## MultipartHttpServletRequest接口定义

```java
/**
 * Provides additional methods for dealing with multipart content within a
 * servlet request, allowing to access uploaded files.
 * Implementations also need to override the standard
 * {@link javax.servlet.ServletRequest} methods for parameter access, making
 * multipart parameters available.
 *
 * <p>A concrete implementation is 具体实现
 * {@link org.springframework.web.multipart.support.DefaultMultipartHttpServletRequest}.
 * As an intermediate step,
 * {@link org.springframework.web.multipart.support.AbstractMultipartHttpServletRequest} 抽象基类
 * can be subclassed.
 *
 * @author Juergen Hoeller
 * @author Trevor D. Cook
 * @since 29.09.2003
 * @see MultipartResolver
 * @see MultipartFile
 * @see javax.servlet.http.HttpServletRequest#getParameter
 * @see javax.servlet.http.HttpServletRequest#getParameterNames
 * @see javax.servlet.http.HttpServletRequest#getParameterMap
 * @see org.springframework.web.multipart.support.DefaultMultipartHttpServletRequest
 * @see org.springframework.web.multipart.support.AbstractMultipartHttpServletRequest
 */
public interface MultipartHttpServletRequest extends HttpServletRequest, MultipartRequest {

	/**
	 * Return this request's method as a convenient HttpMethod instance.
	 */
	@Nullable
	HttpMethod getRequestMethod(); //返回请求方法作为一个便捷的HttpMethod实例：GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE;

	/**
	 * Return this request's headers as a convenient HttpHeaders instance.
	 */
	HttpHeaders getRequestHeaders(); //返回请求头信息作为便捷的HttpHeaders实例

	/**
	 * Return the headers associated 关联 with the specified part of the multipart request.   请求部分
	 * <p>If the underlying implementation supports access to headers, then all headers are returned. 如底层属性支持访问头信息时返回全部，否则至少返回包含Content-Type的头
	 * Otherwise, the returned headers will include a 'Content-Type' header at the very least.
	 */
	@Nullable
	HttpHeaders getMultipartHeaders(String paramOrFileName);//

}
```

## AbstractMultipartHttpServletRequest

抽象基类实现MultipartHttpServletRequest接口，提供管理提前生成MultipartFile接口实例【具体查看：[Spring源码-MultipartFile上传文件接口](/2019/08/08/spring-MultipartFile/)】

`AbstractMultipartHttpServlRequest继承了来自Servlet提供的HttpServletRequestWrapper类，即当前抽象类中可以不需要实现HttpServletRequest接口中定义的方法，仅需要实现MultipartHttpServletRequest接口中的方法`

```java
/**
 * Abstract base implementation of the MultipartHttpServletRequest interface.
 * Provides management of pre-generated提前生成 MultipartFile instances.
 *
 */
public abstract class AbstractMultipartHttpServletRequest extends HttpServletRequestWrapper
		implements MultipartHttpServletRequest {

	@Nullable
	private MultiValueMap<String, MultipartFile> multipartFiles;//多值映射 filed:1-> value:n 


	/**
	 * Wrap the given HttpServletRequest in a MultipartHttpServletRequest. 将给定的HttpServletRequest包装在MultipartHttpServletRequest中
	 * @param request the request to wrap
	 */
	protected AbstractMultipartHttpServletRequest(HttpServletRequest request) {
		super(request);
	}


	@Override
	public HttpServletRequest getRequest() {
		return (HttpServletRequest) super.getRequest();
	}

	@Override
	public HttpMethod getRequestMethod() {
		return HttpMethod.resolve(getRequest().getMethod()); //解析请求方法
	}

	@Override
	public HttpHeaders getRequestHeaders() {
		HttpHeaders headers = new HttpHeaders();
		Enumeration<String> headerNames = getHeaderNames();
		while (headerNames.hasMoreElements()) {
			String headerName = headerNames.nextElement();
			headers.put(headerName, Collections.list(getHeaders(headerName)));
		}
		return headers;
	}

	@Override
	public Iterator<String> getFileNames() {
		return getMultipartFiles().keySet().iterator();
	}

	@Override
	public MultipartFile getFile(String name) {
		return getMultipartFiles().getFirst(name);
	}

	@Override
	public List<MultipartFile> getFiles(String name) {
		List<MultipartFile> multipartFiles = getMultipartFiles().get(name);
		if (multipartFiles != null) {
			return multipartFiles;
		}
		else {
			return Collections.emptyList();
		}
	}

	@Override
	public Map<String, MultipartFile> getFileMap() {
		return getMultipartFiles().toSingleValueMap();
	}

	@Override
	public MultiValueMap<String, MultipartFile> getMultiFileMap() {
		return getMultipartFiles();
	}

	/**
	 * Determine whether the underlying multipart request has been resolved. 确认底层的multipart请求是否已经被解析
	 * @return {@code true} when eagerly initialized立即初始化 or lazily triggered延迟触发,
	 * {@code false} in case of a lazy-resolution request that got aborted
	 * before any parameters or multipart files have been accessed
	 * @since 4.3.15
	 * @see #getMultipartFiles()
	 */
	public boolean isResolved() {
		return (this.multipartFiles != null);
	}


	/**
	 * Set a Map with parameter names as keys and list of MultipartFile objects as values. 设置参数名与列表MultipartFile对象映射
	 * To be invoked by subclasses on initialization. 在初始化时由子类调用
	 */
	protected final void setMultipartFiles(MultiValueMap<String, MultipartFile> multipartFiles) {
		this.multipartFiles =
				new LinkedMultiValueMap<>(Collections.unmodifiableMap(multipartFiles));
	}

	/**
	 * Obtain the MultipartFile Map for retrieval, 获取用于检索的MultipartFile映射
	 * lazily initializing it if necessary.
	 * @see #initializeMultipart()
	 */
	protected MultiValueMap<String, MultipartFile> getMultipartFiles() {
		if (this.multipartFiles == null) {
			initializeMultipart();
		}
		return this.multipartFiles;
	}

	/**
	 * Lazily initialize the multipart request, if possible.尽可能的延迟初始化multipart请求，
	 * Only called if not already eagerly initialized.只有在没有理解初始化时调用
	 */
	protected void initializeMultipart() {
		throw new IllegalStateException("Multipart request not initialized");
	}

}
```

### StandardMultipartHttpServletRequest

Spring的MultipartHttpServletRequest适配器，包装一个Servlet3.+的HttpServletRequest和它的javax.servlet.http.Part对象。参数通过原生的request's getParameter方法(无任何定制处理)。

`由StandardServletMultipartResolver解析HttpServletRequest创建`



```java
/**
 * Spring MultipartHttpServletRequest adapter, wrapping a Servlet 3.0 HttpServletRequest
 * and its Part objects. Parameters get exposed through the native 原生 request's getParameter
 * methods - without any custom processing on our side.
 *
 * @author Juergen Hoeller
 * @author Rossen Stoyanchev
 * @since 3.1
 * @see StandardServletMultipartResolver
 */
public class StandardMultipartHttpServletRequest extends AbstractMultipartHttpServletRequest {

	@Nullable
	private Set<String> multipartParameterNames;//多部分请求参数名


	/**
	 * Create a new StandardMultipartHttpServletRequest wrapper for the given request,
	 * immediately parsing the multipart content. 立即解析多部分内容
	 * @param request the servlet request to wrap
	 * @throws MultipartException if parsing failed
	 */
	public StandardMultipartHttpServletRequest(HttpServletRequest request) throws MultipartException {
		this(request, false);
	}

	/**
	 * Create a new StandardMultipartHttpServletRequest wrapper for the given request.
	 * @param request the servlet request to wrap
	 * @param lazyParsing whether multipart parsing should be triggered lazily  on
	 * first access of multipart files or parameters 首次访问文件或者参数是否延迟触发
	 * @throws MultipartException if an immediate parsing attempt failed
	 * @since 3.2.9
	 */
	public StandardMultipartHttpServletRequest(HttpServletRequest request, boolean lazyParsing)
			throws MultipartException {

		super(request);
		if (!lazyParsing) { //是否延迟解析
			parseRequest(request); //解析请求
		}
	}


	private void parseRequest(HttpServletRequest request) {
		try {
			Collection<Part> parts = request.getParts();
			this.multipartParameterNames = new LinkedHashSet<>(parts.size());//参数名数量
			MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>(parts.size());
			for (Part part : parts) {
				String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION); //Content-Disposition
				ContentDisposition disposition = ContentDisposition.parse(headerValue);
				String filename = disposition.getFilename();//文件名
				if (filename != null) {
					if (filename.startsWith("=?") && filename.endsWith("?=")) { //java email api
						filename = MimeDelegate.decode(filename);
					}
					files.add(part.getName(), new StandardMultipartFile(part, filename));  //StandardMultipartFile
				}
				else {
					this.multipartParameterNames.add(part.getName());
				}
			}
			setMultipartFiles(files); //初始化设置multipartFiles
		}
		catch (Throwable ex) {
			handleParseFailure(ex);//解析失败处理
		}
	}

	protected void handleParseFailure(Throwable ex) {
		String msg = ex.getMessage();
		if (msg != null && msg.contains("size") && msg.contains("exceed")) {
			throw new MaxUploadSizeExceededException(-1, ex);
		}
		throw new MultipartException("Failed to parse multipart servlet request", ex);
	}

	@Override
	protected void initializeMultipart() { //初始化Multipart请求
		parseRequest(getRequest());
	}

	@Override
	public Enumeration<String> getParameterNames() { //参数名
		if (this.multipartParameterNames == null) {  未初始化时进行初始化
			initializeMultipart();
		}
		if (this.multipartParameterNames.isEmpty()) {
			return super.getParameterNames();
		}
		Servlet 3.0 环境getParameterNames() 不能确保包含multipart表单的元素
		// Servlet 3.0 getParameterNames() not guaranteed to include multipart form items
		 为了安全起见，需要把它们合并到这里
		// (e.g. on WebLogic 12) -> need to merge them here to be on the safe side
		Set<String> paramNames = new LinkedHashSet<>();
		Enumeration<String> paramEnum = super.#getParameterNames(); //ServletRequest#getParameterNames参数名+part.getName()
		while (paramEnum.hasMoreElements()) {
			paramNames.add(paramEnum.nextElement());
		}
		paramNames.addAll(this.multipartParameterNames);  //合并初始化参数名，根据HttpServletRequest#getParts()、javax.servlet.http.Part#getName()
		return Collections.enumeration(paramNames);
	}

	@Override
	public Map<String, String[]> getParameterMap() {
		if (this.multipartParameterNames == null) {
			initializeMultipart();
		}
		if (this.multipartParameterNames.isEmpty()) {
			return super.getParameterMap();
		}

		// Servlet 3.0 getParameterMap() not guaranteed to include multipart form items
		// (e.g. on WebLogic 12) -> need to merge them here to be on the safe side
		Map<String, String[]> paramMap = new LinkedHashMap<>();
		paramMap.putAll(super.getParameterMap());
		for (String paramName : this.multipartParameterNames) {
			if (!paramMap.containsKey(paramName)) {
				paramMap.put(paramName, getParameterValues(paramName));
			}
		}
		return paramMap;
	}

	@Override
	public String getMultipartContentType(String paramOrFileName) { //Multipart请求Context-type
		try {
			Part part = getPart(paramOrFileName); //参数名获取Path对象
			return (part != null ? part.getContentType() : null);
		}
		catch (Throwable ex) {
			throw new MultipartException("Could not access multipart servlet request", ex);
		}
	}

	@Override
	public HttpHeaders getMultipartHeaders(String paramOrFileName) {
		try {
			Part part = getPart(paramOrFileName);
			if (part != null) {
				HttpHeaders headers = new HttpHeaders();
				for (String headerName : part.getHeaderNames()) { //Part对象中获取请求头名
					headers.put(headerName, new ArrayList<>(part.getHeaders(headerName)));
				}
				return headers;
			}
			else {
				return null;
			}
		}
		catch (Throwable ex) {
			throw new MultipartException("Could not access multipart servlet request", ex);
		}
	}

	/**
	 * Inner class to avoid a hard dependency on the JavaMail API.
	 */
	private static class MimeDelegate {

		public static String decode(String value) {
			try {
				return MimeUtility.decodeText(value);
			}
			catch (UnsupportedEncodingException ex) {
				throw new IllegalStateException(ex);
			}
		}
	}

}

```

## DefaultMultipartHttpServletRequest

默认MultipartHttpServletRequest接口实现，提供管理提前生产参数值。`由CommonsMultipartResolver解析HttpServletRequest生成`


```java
/**
 * Default implementation of the
 * {@link org.springframework.web.multipart.MultipartHttpServletRequest}
 * interface. Provides management of pre-generated parameter values.
 *
 * <p>Used by  {@link org.springframework.web.multipart.commons.CommonsMultipartResolver}. 
 *
 * @see org.springframework.web.multipart.MultipartResolver
 */
public class DefaultMultipartHttpServletRequest extends AbstractMultipartHttpServletRequest {

	private static final String CONTENT_TYPE = "Content-Type";

	@Nullable
	private Map<String, String[]> multipartParameters; //参数

	@Nullable
	private Map<String, String> multipartParameterContentTypes;//Content-type


	/**
	 * Wrap the given HttpServletRequest in a MultipartHttpServletRequest.
	 * @param request the servlet request to wrap 
	 * @param mpFiles a map of the multipart files 文件
	 * @param mpParams a map of the parameters to expose,
	 * with Strings as keys and String arrays as values
	 */
	public DefaultMultipartHttpServletRequest(HttpServletRequest request, MultiValueMap<String, MultipartFile> mpFiles,
			Map<String, String[]> mpParams, Map<String, String> mpParamContentTypes) {

		super(request);
		setMultipartFiles(mpFiles);//文件
		setMultipartParameters(mpParams);//参数
		setMultipartParameterContentTypes(mpParamContentTypes); //ContentType
	}

	/**
	 * Wrap the given HttpServletRequest in a MultipartHttpServletRequest.
	 * @param request the servlet request to wrap
	 */
	public DefaultMultipartHttpServletRequest(HttpServletRequest request) {
		super(request);
	}


	@Override
	@Nullable
	public String getParameter(String name) { //参数名对应的值
		String[] values = getMultipartParameters().get(name);
		if (values != null) {
			return (values.length > 0 ? values[0] : null);
		}
		return super.getParameter(name);
	}

	@Override
	public String[] getParameterValues(String name) {//参数对应值
		String[] values = getMultipartParameters().get(name);
		if (values != null) {
			return values;
		}
		return super.getParameterValues(name);
	}

	@Override
	public Enumeration<String> getParameterNames() { //参数名
		Map<String, String[]> multipartParameters = getMultipartParameters();
		if (multipartParameters.isEmpty()) {
			return super.getParameterNames();
		}

		Set<String> paramNames = new LinkedHashSet<>();
		Enumeration<String> paramEnum = super.getParameterNames();
		while (paramEnum.hasMoreElements()) {
			paramNames.add(paramEnum.nextElement());
		}
		paramNames.addAll(multipartParameters.keySet()); //合并
		return Collections.enumeration(paramNames);
	}

	@Override
	public Map<String, String[]> getParameterMap() { //参数映射
		Map<String, String[]> multipartParameters = getMultipartParameters();
		if (multipartParameters.isEmpty()) {
			return super.getParameterMap();
		}

		Map<String, String[]> paramMap = new LinkedHashMap<>();
		paramMap.putAll(super.getParameterMap());
		paramMap.putAll(multipartParameters);
		return paramMap;
	}

	@Override
	public String getMultipartContentType(String paramOrFileName) { //Content-Type
		MultipartFile file = getFile(paramOrFileName);
		if (file != null) { //文件Content-Type
			return file.getContentType();
		}
		else {
			return getMultipartParameterContentTypes().get(paramOrFileName); //Content-Type 参数名值
		}
	}

	@Override
	public HttpHeaders getMultipartHeaders(String paramOrFileName) {
		String contentType = getMultipartContentType(paramOrFileName);
		if (contentType != null) {
			HttpHeaders headers = new HttpHeaders();
			headers.add(CONTENT_TYPE, contentType); //请求头信息：Content-Type
			return headers;
		}
		else {
			return null;
		}
	}


	/**
	 * 设置一个映射，参数名作为键，字符串数组对象作为值 
	 * Set a Map with parameter names as keys and String array objects as values.
	 * To be invoked by subclasses on initialization. 在初始化时由子类调用
	 */
	protected final void setMultipartParameters(Map<String, String[]> multipartParameters) {
		this.multipartParameters = multipartParameters;
	}

	/**
	 * Obtain the multipart parameter Map for retrieval,获取用于检索的multipart 请求参数映射
	 * lazily initializing it if necessary.如果需要延迟初始化
	 * @see #initializeMultipart()
	 */
	protected Map<String, String[]> getMultipartParameters() {
		if (this.multipartParameters == null) {
			initializeMultipart();
		}
		return this.multipartParameters;
	}

	/**
	 * Set a Map with parameter names as keys and content type Strings as values. 设置一个映射，参数名作为键，内容类型字符串作为值
	 * To be invoked by subclasses on initialization. 在初始化时由子类调用
	 */
	protected final void setMultipartParameterContentTypes(Map<String, String> multipartParameterContentTypes) {
		this.multipartParameterContentTypes = multipartParameterContentTypes;
	}

	/**
	 * Obtain the multipart parameter content type Map for retrieval, 获取用于检索的多部分参数内容类型映射
	 * lazily initializing it if necessary. 如果需要，可以延迟初始化它
	 * @see #initializeMultipart()
	 */
	protected Map<String, String> getMultipartParameterContentTypes() {
		if (this.multipartParameterContentTypes == null) {
			initializeMultipart();
		}
		return this.multipartParameterContentTypes;
	}

}
```
