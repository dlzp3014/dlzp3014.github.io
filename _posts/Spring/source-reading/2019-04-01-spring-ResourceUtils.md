---
layout: post
title:  "Spring源码-ResourceUtils"
date:   2019-04-01 23:42:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}

ResourceUtils主要用于解析系统文件中位置资源的文件，用于框架内部

## 类描述

```java
/**
 * Utility methods for resolving resource locations to files in the
 * file system. Mainly for internal use within the framework. 
 *
 * <p>Consider using Spring's Resource abstraction in the core package 考虑到Spring的资源抽象
 * for handling all kinds of file resources in a uniform manner(用于以统一的方式处理各种文件资源).
 * {@link org.springframework.core.io.ResourceLoader}'s {@code getResource()}  ResourceLoader的getResource方法能够解析任何位置为Resource对象
 * method can resolve any location to a {@link org.springframework.core.io.Resource}
 * object, which in turn allows one to obtain a {@code java.io.File} in the 包含获取文件通过getFile方法
 * file system through its {@code getFile()} method.
 *
 * @author Juergen Hoeller
 * @since 1.1.5
 * @see org.springframework.core.io.Resource
 * @see org.springframework.core.io.ClassPathResource
 * @see org.springframework.core.io.FileSystemResource
 * @see org.springframework.core.io.UrlResource
 * @see org.springframework.core.io.ResourceLoader
 */
public abstract class ResourceUtils {

}
```






## 常量

```java
/** Pseudo URL prefix for loading from the class path: "classpath:" */
	public static final String CLASSPATH_URL_PREFIX = "classpath:";

	/** URL prefix for loading from the file system: "file:" */
	public static final String FILE_URL_PREFIX = "file:";

	/** URL prefix for loading from a jar file: "jar:" */
	public static final String JAR_URL_PREFIX = "jar:";

	/** URL prefix for loading from a war file on Tomcat: "war:" */
	public static final String WAR_URL_PREFIX = "war:";

	/** URL protocol for a file in the file system: "file" */
	public static final String URL_PROTOCOL_FILE = "file";

	/** URL protocol for an entry from a jar file: "jar" */
	public static final String URL_PROTOCOL_JAR = "jar";

	/** URL protocol for an entry from a war file: "war" */
	public static final String URL_PROTOCOL_WAR = "war";

	/** URL protocol for an entry from a zip file: "zip" */
	public static final String URL_PROTOCOL_ZIP = "zip";

	/** URL protocol for an entry from a WebSphere jar file: "wsjar" */
	public static final String URL_PROTOCOL_WSJAR = "wsjar";

	/** URL protocol for an entry from a JBoss jar file: "vfszip" */
	public static final String URL_PROTOCOL_VFSZIP = "vfszip";

	/** URL protocol for a JBoss file system resource: "vfsfile" */
	public static final String URL_PROTOCOL_VFSFILE = "vfsfile";

	/** URL protocol for a general JBoss VFS resource: "vfs" */
	public static final String URL_PROTOCOL_VFS = "vfs";

	/** File extension for a regular jar file: ".jar" */
	public static final String JAR_FILE_EXTENSION = ".jar";

	/** Separator between JAR URL and file path within the JAR: "!/" */ jar url 和文件分割符，如jar:file:/E:\m2\repository\mysql\mysql-connector-java\8.0.12\mysql-connector-java-8.0.12.jar!/META-INF/INDEX.LIST
	public static final String JAR_URL_SEPARATOR = "!/";

	/** Special separator between WAR URL and jar part on Tomcat */
	public static final String WAR_URL_SEPARATOR = "*/";

```

## 核心方法


- boolean isUrl(@Nullable String resourceLocation)：是否为url

```java
/**
 * Return whether the given resource location is a URL: 给定的位置资源是否为URL
 * either a special "classpath" pseudo URL or a standard URL. 任何一个指定的classpath 伪装的URL或者标准的URL
 * @param resourceLocation the location String to check
 * @return whether the location qualifies as a URL
 * @see #CLASSPATH_URL_PREFIX
 * @see java.net.URL
 */
public static boolean isUrl(@Nullable String resourceLocation) {
	if (resourceLocation == null) {
		return false;
	}
	if (resourceLocation.startsWith(CLASSPATH_URL_PREFIX)) { //以classpath开头
		return true;
	}
	try {
		new URL(resourceLocation); //构建URL,出现异常返回false
		return true;
	}
	catch (MalformedURLException ex) {
		return false;
	}
}
```

- URL getURL(String resourceLocation)：资源路径解析为URL


```java
/**
 * Resolve the given resource location to a {@code java.net.URL}. 解析给定的资源位置为URL,不检查URL是否真实存在
 * <p>Does not check whether the URL actually exists; simply returns
 * the URL that the given location would correspond to. 符合要求
 * @param resourceLocation the resource location to resolve: either a
 * "classpath:" pseudo URL, a "file:" URL, or a plain file path
 * @return a corresponding URL object
 * @throws FileNotFoundException if the resource cannot be resolved to a URL
 */
public static URL getURL(String resourceLocation) throws FileNotFoundException {
	Assert.notNull(resourceLocation, "Resource location must not be null");
	if (resourceLocation.startsWith(CLASSPATH_URL_PREFIX)) { 以classpath开头时使用默认的累加器加载资源
		String path = resourceLocation.substring(CLASSPATH_URL_PREFIX.length());
		ClassLoader cl = ClassUtils.getDefaultClassLoader();
		URL url = (cl != null ? cl.getResource(path) : ClassLoader.getSystemResource(path));
		if (url == null) {
			String description = "class path resource [" + path + "]";
			throw new FileNotFoundException(description +
					" cannot be resolved to URL because it does not exist");
		}
		return url;
	}
	try {
		// try URL 尝试构建URL
		return new URL(resourceLocation);
	}
	catch (MalformedURLException ex) {
		// no URL -> treat as file path 视为文件路径
		try {
			return new File(resourceLocation).toURI().toURL(); 
		}
		catch (MalformedURLException ex2) {
			throw new FileNotFoundException("Resource location [" + resourceLocation +
					"] is neither a URL not a well-formed file path");
		}
	}
}
```

- File getFile(String resourceLocation)：资源路径解析为File，可以为"classpath:"或者"file:"的URL

类似方法：


File getFile(URI resourceUri)/File getFile(URI resourceUri, String description):从URI中解析为File


File getFile(URL resourceUrl)/File getFile(URL resourceUrl, String description):从URL中解析为File




```java
/**
 * Resolve the given resource location to a {@code java.io.File}, 解析资源路径为文件
 * i.e. to a file in the file system.
 * <p>Does not check whether the file actually exists; simply returns 不检查文件是否真实存在，简单返回符合给定路径的File
 * the File that the given location would correspond to.
 * @param resourceLocation the resource location to resolve: either a 解析资源路径：classpath：或file：、前缀或者普通文件路径
 * "classpath:" pseudo URL, a "file:" URL, or a plain file path
 * @return a corresponding File object
 * @throws FileNotFoundException if the resource cannot be resolved to
 * a file in the file system
 */
public static File getFile(String resourceLocation) throws FileNotFoundException {
	Assert.notNull(resourceLocation, "Resource location must not be null");
	if (resourceLocation.startsWith(CLASSPATH_URL_PREFIX)) { //以classpath开头截取，获取默认的类加载器加载文件
		String path = resourceLocation.substring(CLASSPATH_URL_PREFIX.length());
		String description = "class path resource [" + path + "]";
		ClassLoader cl = ClassUtils.getDefaultClassLoader();
		URL url = (cl != null ? cl.getResource(path) : ClassLoader.getSystemResource(path)); //使用类加载器加载资源返回URL
		if (url == null) {
			throw new FileNotFoundException(description +
					" cannot be resolved to absolute file path because it does not exist");
		}
		return getFile(url, description); //文件
	}
	try {
		// try URL
		return getFile(new URL(resourceLocation));
	}
	catch (MalformedURLException ex) {
		// no URL -> treat as file path
		return new File(resourceLocation);
	}
}

/**
 * Resolve the given resource URL to a {@code java.io.File}, 解析给定的资源URL为一个File
 * i.e. to a file in the file system. 如为一个文件系统的文件
 * @param resourceUrl the resource URL to resolve 资源url用于解析
 * @return a corresponding File object
 * @throws FileNotFoundException if the URL cannot be resolved to
 * a file in the file system
 */
public static File getFile(URL resourceUrl) throws FileNotFoundException {
	return getFile(resourceUrl, "URL");
}

/**
 * Resolve the given resource URL to a {@code java.io.File},
 * i.e. to a file in the file system.
 * @param resourceUrl the resource URL to resolve
 * @param description a description of the original resource that URL资源已被创建的原始资源描述
 * the URL was created for (for example, a class path location)
 * @return a corresponding File object
 * @throws FileNotFoundException if the URL cannot be resolved to URL不能被解析为一个文件系统的文件
 * a file in the file system
 */
public static File getFile(URL resourceUrl, String description) throws FileNotFoundException {
	Assert.notNull(resourceUrl, "Resource URL must not be null");
	if (!URL_PROTOCOL_FILE.equals(resourceUrl.getProtocol())) { //URL协议
		throw new FileNotFoundException(
				description + " cannot be resolved to absolute file path " +
				"because it does not reside in the file system: " + resourceUrl);
	}
	try {
		return new File(toURI(resourceUrl).getSchemeSpecificPart()); //URL转换为URI,(URI#getSchemeSpecificPart,Returns the decoded scheme-specific part of this URI)
	}
	catch (URISyntaxException ex) {
		// Fallback for URLs that are not valid URIs (should hardly ever happen几乎不应该发生).
		return new File(resourceUrl.getFile());
	}
}

/**
 * Resolve the given resource URI to a {@code java.io.File}, 解析URI为File
 * i.e. to a file in the file system.
 * @param resourceUri the resource URI to resolve
 * @return a corresponding File object
 * @throws FileNotFoundException if the URL cannot be resolved to
 * a file in the file system
 * @since 2.5
 */
public static File getFile(URI resourceUri) throws FileNotFoundException {
	return getFile(resourceUri, "URI");
}

/**
 * Resolve the given resource URI to a {@code java.io.File},
 * i.e. to a file in the file system.
 * @param resourceUri the resource URI to resolve
 * @param description a description of the original resource that
 * the URI was created for (for example, a class path location)
 * @return a corresponding File object
 * @throws FileNotFoundException if the URL cannot be resolved to
 * a file in the file system
 * @since 2.5
 */
public static File getFile(URI resourceUri, String description) throws FileNotFoundException {
	Assert.notNull(resourceUri, "Resource URI must not be null");
	if (!URL_PROTOCOL_FILE.equals(resourceUri.getScheme())) { //获取Scheme
		throw new FileNotFoundException(
				description + " cannot be resolved to absolute file path " +
				"because it does not reside in the file system: " + resourceUri);
	}
	return new File(resourceUri.getSchemeSpecificPart());
}
```
- boolean isFileURL(URL url):URL是否为文件

相识方法：

isJarURL(URL url)：URL使用为jar文件资源


boolean isJarFileURL(URL url)：

```java
/**
 * Determine whether the given URL points to a resource in the file system, URL指向系统文件资源
 * i.e. has protocol "file", "vfsfile" or "vfs".
 * @param url the URL to check
 * @return whether the URL has been identified as a file system URL
 */
public static boolean isFileURL(URL url) {
	String protocol = url.getProtocol();
	return (URL_PROTOCOL_FILE.equals(protocol) || URL_PROTOCOL_VFSFILE.equals(protocol) ||
			URL_PROTOCOL_VFS.equals(protocol));
}

/**
 * Determine whether the given URL points to a resource in a jar file.
 * i.e. has protocol "jar", "war, ""zip", "vfszip" or "wsjar".
 * @param url the URL to check
 * @return whether the URL has been identified as a JAR URL
 */
public static boolean isJarURL(URL url) {
	String protocol = url.getProtocol();
	return (URL_PROTOCOL_JAR.equals(protocol) || URL_PROTOCOL_WAR.equals(protocol) ||
			URL_PROTOCOL_ZIP.equals(protocol) || URL_PROTOCOL_VFSZIP.equals(protocol) ||
			URL_PROTOCOL_WSJAR.equals(protocol));
}

/**
 * Determine whether the given URL points to a jar file itself,
 * that is, has protocol "file" and ends with the ".jar" extension. 协议为"file:"并且路径以".jar"结尾
 * @param url the URL to check
 * @return whether the URL has been identified as a JAR file URL
 * @since 4.1
 */
public static boolean isJarFileURL(URL url) {
	return (URL_PROTOCOL_FILE.equals(url.getProtocol()) &&
			url.getPath().toLowerCase().endsWith(JAR_FILE_EXTENSION));
}

```

- URL extractJarFileURL(URL jarUrl)：提取URL中真实的jar URL

 URL extractArchiveURL(URL jarUrl): 从给定的JAR/war URL中提交最外层的URL


```java

/**
 * Extract the URL for the actual jar file from the given URL 从给定的URL中提取实际jar文件的URL
 * (which may point to a resource in a jar file or to a jar file itself).
 * @param jarUrl the original URL
 * @return the URL for the actual jar file
 * @throws MalformedURLException if no valid jar file URL could be extracted 如果不能提取有效的jar文件URL
 */
public static URL extractJarFileURL(URL jarUrl) throws MalformedURLException {
	String urlFile = jarUrl.getFile();
	int separatorIndex = urlFile.indexOf(JAR_URL_SEPARATOR); //jar文件分隔符
	if (separatorIndex != -1) {
		String jarFile = urlFile.substring(0, separatorIndex); //截取前半部分，即"xxx/xxx.jar!/xxx/xxx.text" 返回""xxx/xxx.jar"
		try {
			return new URL(jarFile);
		}
		catch (MalformedURLException ex) {
			// Probably no protocol in original jar URL, like "jar:C:/mypath/myjar.jar". 可能在原始jar URL中没有协议
			// This usually indicates(这通常表示) that the jar file resides in the file system. jar文件存在于系统文件中
			if (!jarFile.startsWith("/")) { 
				jarFile = "/" + jarFile;
			}
			return new URL(FILE_URL_PREFIX + jarFile);
		}
	}
	else {
		return jarUrl;
	}
}

/**
 * Extract the URL for the outermost archive from the given jar/war URL 从给定的JAR/war URL中提交最外层的URL
 * (which may point to a resource in a jar file or to a jar file itself).
 * <p>In the case of a jar file nested within a war file, this will return 在一个war文件中内部的一个JarFile的情况下，将返回指向war文件的URL,因为这是文件系统中可解析的URL
 * a URL to the war file since that is the one resolvable in the file system.
 * @param jarUrl the original URL
 * @return the URL for the actual jar file
 * @throws MalformedURLException if no valid jar file URL could be extracted
 * @since 4.1.8
 * @see #extractJarFileURL(URL)
 */
public static URL extractArchiveURL(URL jarUrl) throws MalformedURLException {
	String urlFile = jarUrl.getFile();

	int endIndex = urlFile.indexOf(WAR_URL_SEPARATOR); //war包分割符
	if (endIndex != -1) {
		// Tomcat's "war:file:...mywar.war*/WEB-INF/lib/myjar.jar!/myentry.txt"
		String warFile = urlFile.substring(0, endIndex); //截取前半部分
		if (URL_PROTOCOL_WAR.equals(jarUrl.getProtocol())) { //是否已WAR:协议
			return new URL(warFile);
		}
		int startIndex = warFile.indexOf(WAR_URL_PREFIX); 
		if (startIndex != -1) {
			return new URL(warFile.substring(startIndex + WAR_URL_PREFIX.length())); //截取war包协议后半部分
		}
	}

	// Regular "jar:file:...myjar.jar!/myentry.txt"
	return extractJarFileURL(jarUrl);
}


```

-  URI toURI(URL url)/URI toURI(String location): URL/String转换为URI，首先用“%20”URI编码替换空格


``` java
/**
 * Create a URI instance for the given URL, 从给定的URL创建一个URI实例
 * replacing spaces with "%20" URI encoding first. 首先用“%20”URI编码替换空格
 * @param url the URL to convert into a URI instance url转换为URI实例的URL
 * @return the URI instance
 * @throws URISyntaxException if the URL wasn't a valid URI
 * @see java.net.URL#toURI()
 */
public static URI toURI(URL url) throws URISyntaxException {
	return toURI(url.toString());
}

/**
 * Create a URI instance for the given location String,从给定的文件创建URI实例
 * replacing spaces with "%20" URI encoding first. 
 * @param location the location String to convert into a URI instance
 * @return the URI instance
 * @throws URISyntaxException if the location wasn't a valid URI
 */
public static URI toURI(String location) throws URISyntaxException {
	return new URI(StringUtils.replace(location, " ", "%20"));
}

```

- useCachesIfNecessary(URLConnection con) 

```java
/**
 * Set the {@link URLConnection#setUseCaches "useCaches"} flag on the 从给定的connection设置useCaches的标记
 * given connection, preferring {@code false}(首选false) but leaving the 
 * flag at {@code true} for JNLP based resources. 仅使用基于JNLP的资源为true
 * @param con the URLConnection to set the flag on
 */
public static void useCachesIfNecessary(URLConnection con) {
	con.setUseCaches(con.getClass().getSimpleName().startsWith("JNLP"));
}

```