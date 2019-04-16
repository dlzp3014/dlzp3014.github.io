---
layout: post
title:  "Spring源码-Resource资源抽象"
date:   2019-03-20 21:19:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}


Resource接口提供了更强大的底层资源访问能力，该接口拥有对应不同资源类型的实现类。具体使用过程参考：[【Spring中资源访问】](/2019/04/11/spring-Resource-access/)









## 接口继承关系


![](/img/post.img/spring/resource-interface.png)



- InputStreamSource：输入流源

```java
package org.springframework.core.io;

/**
 * Simple interface for objects that are sources for an {@link InputStream}. 作为InputStream源的简单对象接口
 *
 * <p>This is the base interface for Spring's more extensive(广泛、大量) {@link Resource} interface. InputStreamSource接口是 Resource接口的基本接口
 *
 * <p>For single-use streams, {@link InputStreamResource} can be used for any 对于一次性的流，InputStreamResource可以使用到任何给定的InputStream中，
 * given {@code InputStream}. Spring's {@link ByteArrayResource} or any Spring的ByteArrayResource或者基于文件的Resource实现可作为具体的实例
 * file-based {@code Resource} implementation can be used as a concrete
 * instance, allowing one to read the underlying content stream multiple times. 允许用户多次读取底层流内容
 * This makes this interface useful (这使得当前接口非常有用) as an abstract content source for mail
 * attachments, for example.
 *
 * @author Juergen Hoeller
 * @since 20.01.2004
 * @see java.io.InputStream
 * @see Resource
 * @see InputStreamResource
 * @see ByteArrayResource
 */
public interface InputStreamSource {

	/**
	 * Return an {@link InputStream} for the content of an underlying resource. 返回底层资源内容的一个InputStream
	 * <p>It is expected（预期） that each call creates a <i>fresh</i> stream.  每次调用创建一个新的流
	 * <p>This requirement is particularly important(这个要求特别重要) when you consider(考虑) an API such
	 * as JavaMail, which needs to be able to read the stream multiple times(能够多次读取流) when
	 * creating mail attachments. For such a use case(对于这样的用例), it is <i>required</i>
	 * that each {@code getInputStream()} call returns a fresh stream.
	 * @return the input stream for the underlying resource (must not be {@code null})
	 * @throws java.io.FileNotFoundException if the underlying resource doesn't exist
	 * @throws IOException if the content stream could not be opened
	 */
	InputStream getInputStream() throws IOException;

}

```

- Resource：核心资源接口

```java

package org.springframework.core.io;


/**
 * Interface for a resource descriptor(资源描述接口) that abstracts from the actual 从底层资源的实际类型中抽象出来
 * type of underlying resource, such as a file or class path resource. 一个文件或者类路径资源
 *
 * <p>An InputStream can be opened for every resource(对于任意的resource可以打开一个InputStream) if it exists in 如果它存在于物理形态
 * physical form, but a URL or File handle can just be returned for 一个url或者文件句柄仅仅是返回某些资源
 * certain resources. The actual behavior is implementation-specific.(实际的行为由实现指明)
 *
 * @author Juergen Hoeller
 * @since 28.12.2003
 * @see #getInputStream()
 * @see #getURL()
 * @see #getURI()
 * @see #getFile()
 * @see WritableResource
 * @see ContextResource
 * @see UrlResource
 * @see FileUrlResource
 * @see FileSystemResource
 * @see ClassPathResource
 * @see ByteArrayResource
 * @see InputStreamResource
 */
public interface Resource extends InputStreamSource {

	/**
	 * Determine whether this resource actually exists in physical form.确定此资源是否以物理形式实际存在
	 * <p>This method performs a definitive existence check(确定性存在检查), whereas the
	 * existence of a {@code Resource} handle only guarantees a valid 
	 * descriptor handle.(有效描述)
	 */
	boolean exists();

	/**
	 * Indicate whether the contents of this resource can be read via (资源是否可读)
	 * {@link #getInputStream()}.
	 * <p>Will be {@code true} for typical resource descriptors; 对于典型的资源描述符将返回true
	 * note that actual content reading may still fail when attempted(当尝试).
	 * However, a value of {@code false} is a definitive indication(明确的指示)
	 * that the resource content cannot be read.
	 * @see #getInputStream()
	 */
	default boolean isReadable() {
		return true;
	}

	/**
	 * Indicate whether this resource represents(代表、表示) a handle with an open stream(打开流的句柄）.
	 * If {@code true}, the InputStream cannot be read multiple times, 返回True时，当前InputStream不能多次读取
	 * and must be read and closed to avoid resource leaks(避免资源泄露).
	 * <p>Will be {@code false} for typical resource descriptors.
	 */
	default boolean isOpen() {
		return false;
	}

	/**
	 * Determine whether this resource represents a file in a file system(文件系统中的文件).
	 * A value of {@code true} strongly suggests (but does not guarantee) 强烈建议但不保证
	 * that a {@link #getFile()} call will succeed. (getFile成功调用)
	 * <p>This is conservatively(谨慎地、适当地) {@code false} by default.
	 * @since 5.0
	 * @see #getFile()
	 */
	default boolean isFile() {
		return false;
	}

	/**
	 * Return a URL handle for this resource. 返回此资源的URL句柄
	 * @throws IOException if the resource cannot be resolved as URL(如果当前资源不能解析成一个url), 
	 * i.e. if the resource is not available as descriptor 如果资源不能作为描述符使用
	 */
	URL getURL() throws IOException;

	/**
	 * Return a URI handle for this resource. 返回此资源的URI句柄
	 * @throws IOException if the resource cannot be resolved as URI,
	 * i.e. if the resource is not available as descriptor
	 * @since 2.5
	 */
	URI getURI() throws IOException;

	/**
	 * Return a File handle for this resource. 返回资源的文件句柄
	 * @throws java.io.FileNotFoundException if the resource cannot be resolved as
	 * absolute file path(资源不能被解析为绝对文件路径), i.e. if the resource is not available in a file system(文件系统中不可用)
	 * @throws IOException in case of general resolution/reading failures
	 * @see #getInputStream()
	 */
	File getFile() throws IOException;

	/**
	 * Return a {@link ReadableByteChannel}.  返回可读字节管道
	 * <p>It is expected that each call creates a <i>fresh</i> channel. 每次调用创建一个新的channel
	 * <p>The default implementation returns {@link Channels#newChannel(InputStream)}
	 * with the result of {@link #getInputStream()}. 
	 * @return the byte channel for the underlying resource (must not be {@code null}) 返回底层资源的字节管道
	 * @throws java.io.FileNotFoundException if the underlying resource doesn't exist
	 * @throws IOException if the content channel could not be opened
	 * @since 5.0
	 * @see #getInputStream()
	 */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * Determine the content length for this resource. 资源长度
	 * @throws IOException if the resource cannot be resolved
	 * (in the file system or as some other known physical resource type)
	 */
	long contentLength() throws IOException;

	/**
	 * Determine the last-modified timestamp for this resource. 最后修改时间
	 * @throws IOException if the resource cannot be resolved
	 * (in the file system or as some other known physical resource type)
	 */
	long lastModified() throws IOException;

	/**
	 * Create a resource relative to this resource. 创建当前资源的相对资源
	 * @param relativePath the relative path (relative to this resource 相对当前资源) 相对路径
	 * @return the resource handle for the relative resource
	 * @throws IOException if the relative resource cannot be determined
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * Determine a filename for this resource, i.e. typically the last 确定文件名
	 * part of the path: for example, "myfile.txt".
	 * <p>Returns {@code null} if this type of resource does not
	 * have a filename.
	 */
	@Nullable
	String getFilename();

	/**
	 * Return a description for this resource(返回此资源的描述),
	 * to be used for error output when working with the resource(用于处理资源时的错误输出).
	 * <p>Implementations are also encouraged(鼓励) to return this value
	 * from their {@code toString} method.
	 * @see Object#toString()
	 */
	String getDescription();

}



```

Channels.newChannel(getInputStream()): Java NIO管道 实现如下
```java

    /**
     * Constructs a channel that reads bytes from the given stream. 从给定的流中创建读取字节的管道
     *
     * <p> The resulting channel will not be buffered; it will simply redirect
     * its I/O operations to the given stream.  Closing the channel will in
     * turn cause the stream to be closed.  </p>
     *
     * @param  in
     *         The stream from which bytes are to be read
     *
     * @return  A new readable byte channel
     */
    public static ReadableByteChannel newChannel(final InputStream in) {
        checkNotNull(in, "in");

        if (in instanceof FileInputStream &&
            FileInputStream.class.equals(in.getClass())) {
            return ((FileInputStream)in).getChannel();
        }

        return new ReadableByteChannelImpl(in);
    }

```

- ContextResource：上下文资源

```java
package org.springframework.core.io;

/**
 * Extended interface for a resource that is loaded from an enclosing （封闭context）
 * 'context', e.g. from a {@link javax.servlet.ServletContext} but also
 * from plain classpath paths(普通类路径) or relative file system paths（文件系统相对路径） (specified
 * without an explicit prefix(一个显式的前缀), hence applying relative to the local
 * {@link ResourceLoader}'s context).(本地资源加载器上下文)
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see org.springframework.web.context.support.ServletContextResource
 */
public interface ContextResource extends Resource {

	/**
	 * Return the path within the enclosing 'context'. 返回context封闭内的路径
	 * <p>This is typically path relative to a context-specific root directory(相对于指定上下文的根路径),
	 * e.g. a ServletContext root or a PortletContext root.
	 */
	String getPathWithinContext();

}

```

- WritableResource:可写资源

```java
package org.springframework.core.io;

/**
 * Extended interface for a resource that supports writing to it(支持写入).
 * Provides an {@link #getOutputStream() OutputStream accessor} 提供OutputStream访问器.
 *
 * @author Juergen Hoeller
 * @since 3.1
 * @see java.io.OutputStream
 */
public interface WritableResource extends Resource {

	/**
	 * Indicate whether the contents of this resource can be written 是否可写入
	 * via {@link #getOutputStream()}.
	 * <p>Will be {@code true} for typical resource descriptors;
	 * note that actual content writing may still fail when attempted.
	 * However, a value of {@code false} is a definitive indication false时明确指示当前资源内容不能修改
	 * that the resource content cannot be modified.
	 * @see #getOutputStream()
	 * @see #isReadable()
	 */
	default boolean isWritable() {
		return true;
	}

	/**
	 * Return an {@link OutputStream} for the underlying resource,返回底层资源的一个OutputStream
	 * allowing to (over-)write its content.
	 * @throws IOException if the stream could not be opened
	 * @see #getInputStream()
	 */
	OutputStream getOutputStream() throws IOException;

	/**
	 * Return a {@link WritableByteChannel}. 返回一个可写入字节管道
	 * <p>It is expected that each call creates a <i>fresh</i> channel. 每次调用创建一个新的管道
	 * <p>The default implementation returns {@link Channels#newChannel(OutputStream)}
	 * with the result of {@link #getOutputStream()}.
	 * @return the byte channel for the underlying resource (must not be {@code null})
	 * @throws java.io.FileNotFoundException if the underlying resource doesn't exist
	 * @throws IOException if the content channel could not be opened
	 * @since 5.0
	 * @see #getOutputStream()
	 */
	default WritableByteChannel writableChannel() throws IOException {
		return Channels.newChannel(getOutputStream());
	}

}

```

- HttpResource：Http资源

```java

package org.springframework.web.servlet.resource;

/**
 * Extended interface for a {@link Resource} to be written to an 能够写入Http response响应
 * HTTP response. 
 *
 * @author Brian Clozel
 * @since 5.0
 */
public interface HttpResource extends Resource {

	/**
	 * The HTTP headers to be contributed to the HTTP response
	 * that serves the current resource(服务与当前资源).
	 * @return the HTTP response headers
	 */
	HttpHeaders getResponseHeaders();

}

```

## 接口实现

![](/img/post.img/spring/InputStreamSource-01.png)


![](/img/post.img/spring/InputStreamSource-02.png)



### AbstractResource：Resource接口的抽象实现



```java

package org.springframework.core.io;

/**
 * Convenience base class for {@link Resource} implementations(Resource接口遍历的基础实现),
 * pre-implementing typical behavior.
 *
 * <p>The "exists" method will check whether a File or InputStream can(exists方法将检测文件或者输入流能否打开)
 * be opened; "isOpen" will always return false(isOpen方法总是返回false); "getURL" and "getFile"
 * throw an exception; and "toString" will return the description(toString返回资源的描述).
 *
 * @author Juergen Hoeller
 * @since 28.12.2003
 */
public abstract class AbstractResource implements Resource {

	/**
	 * This implementation checks whether a File can be opened, //检查文件是否能打开 
	 * falling back to whether an InputStream can be opened.失败后检查输入流是否能打开
	 * This will cover both directories and content resources. 涵盖目录和资源内容
	 */
	@Override
	public boolean exists() {
		// Try file existence: can we find the file in the file system? 尝试文件存在，文件系统使用存在当前文件
		try {
			return getFile().exists();
		}
		catch (IOException ex) {
			// Fall back to stream existence: can we open the stream?
			try {
				getInputStream().close(); //可关闭，失败后返回false
				return true;
			}
			catch (Throwable isEx) {
				return false;
			}
		}
	}

	/**
	 * This implementation always returns {@code true}. 是否可读：默认实现，总是返回true
	 */
	@Override
	public boolean isReadable() {
		return true;
	}

	/**
	 * This implementation always returns {@code false}. 是否可打开:默认实现总是返回false
	 */
	@Override
	public boolean isOpen() {
		return false;
	}

	/**
	 * This implementation always returns {@code false}.是否为文件：默认实现总是返回false
	 */
	@Override
	public boolean isFile() {
		return false;
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming 返回资源的URL ,默认实现抛出FileNotFoundException
	 * that the resource cannot be resolved to a URL.无法将资源解析为URL
	 */
	@Override
	public URL getURL() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to URL");
	}

	/**
	 * This implementation builds a URI based on the URL returned 从URL中获取URI
	 * by {@link #getURL()}.
	 */
	@Override
	public URI getURI() throws IOException {
		URL url = getURL();
		try {
			return ResourceUtils.toURI(url);
		}
		catch (URISyntaxException ex) {
			throw new NestedIOException("Invalid URI [" + url + "]", ex);
		}
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming 
	 * that the resource cannot be resolved to an absolute file path.资源不能被解析成一个绝对文件路径
	 */
	@Override
	public File getFile() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to absolute file path");
	}

	/**
	 * This implementation returns {@link Channels#newChannel(InputStream)}
	 * with the result of {@link #getInputStream()}.
	 * <p>This is the same as in {@link Resource}'s corresponding default method 与Resource默认方法一直
	 * but mirrored here (这里的镜像为了在类层次结构中的高效jvm调度)for efficient JVM-level dispatching in a class hierarchy. 
	 */
	@Override
	public ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * This implementation reads the entire InputStream to calculate the 计算整个输入流内容的长度
	 * content length. Subclasses will almost always be able to provide 子类通常能提供最优的版本
	 * a more optimal version of this, e.g. checking a File length.
	 * @see #getInputStream()
	 */
	@Override
	public long contentLength() throws IOException {
		InputStream is = getInputStream(); //获取输入流，然后循环读取，累计计算总数
		try {
			long size = 0;
			byte[] buf = new byte[256];
			int read;
			while ((read = is.read(buf)) != -1) {
				size += read;
			}
			return size;
		}
		finally {
			try {
				is.close(); //输入流关闭
			}
			catch (IOException ex) {
			}
		}
	}

	/**
	 * This implementation checks the timestamp of the underlying File,底层文件最后修改的时间戳
	 * if available.
	 * @see #getFileForLastModifiedCheck()
	 */
	@Override
	public long lastModified() throws IOException {
		File fileToCheck = getFileForLastModifiedCheck();
		long lastModified = fileToCheck.lastModified();//直接使用File的lastModified方法
		if (lastModified == 0L && !fileToCheck.exists()) { //时间戳为0或者文件不存在时抛出异常
			throw new FileNotFoundException(getDescription() +
					" cannot be resolved in the file system for checking its last-modified timestamp");
		}
		return lastModified;
	}

	/**
	 * Determine the File to use for timestamp checking. 确定用于时间戳检查的文件
	 * <p>The default implementation delegates to {@link #getFile()}. 默认实现委托给getFile方法
	 * @return the File to use for timestamp checking (never {@code null})
	 * @throws FileNotFoundException if the resource cannot be resolved as
	 * an absolute file path, i.e. is not available in a file system
	 * @throws IOException in case of general resolution/reading failures
	 */
	protected File getFileForLastModifiedCheck() throws IOException {
		return getFile();
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that relative resources(当前资源的相对资源) cannot be created for this resource.
	 */
	@Override
	public Resource createRelative(String relativePath) throws IOException {
		throw new FileNotFoundException("Cannot create a relative resource for " + getDescription());
	}

	/**
	 * This implementation always returns {@code null}, 文件名称
	 * assuming that this resource type does not have a filename.
	 */
	@Override
	@Nullable
	public String getFilename() {
		return null;
	}


	/**
	 * This implementation compares description strings.
	 * @see #getDescription()
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof Resource &&
				((Resource) other).getDescription().equals(getDescription())));
	}

	/**
	 * This implementation returns the description's hash code.
	 * @see #getDescription()
	 */
	@Override
	public int hashCode() {
		return getDescription().hashCode();
	}

	/**
	 * This implementation returns the description of this resource.
	 * @see #getDescription()
	 */
	@Override
	public String toString() {
		return getDescription();
	}

}


```

### AbstractFileResolvingResource：资源文件基类，继承AbstractResource，用于将url解析为文件引用

```java

package org.springframework.core.io;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;
import java.net.URLConnection;
import java.nio.channels.FileChannel;
import java.nio.channels.ReadableByteChannel;
import java.nio.file.NoSuchFileException;
import java.nio.file.StandardOpenOption;

import org.springframework.util.ResourceUtils;

/**
 * Abstract base class for resources(resources基类) which resolve URLs into File references(解析URLS到File引用),
 * such as {@link UrlResource} or {@link ClassPathResource}.
 *
 * <p>Detects the "file" protocol(发现file协议) as well as the JBoss "vfs" protocol in URLs,
 * resolving file system references accordingly(相应地解析文件系统引用).
 *
 * @author Juergen Hoeller
 * @since 3.0
 */
public abstract class AbstractFileResolvingResource extends AbstractResource {

	/**
	*  1.获取URL
	*  2.是否为File协议的url，如果是获取file，检查是否存在
	*  3.使用HttpURLConnection检查URL连接是否存在
	*  4.自由定义Connection检查
	*  5.使用InputStream是否可关闭
	*/
	@Override
	public boolean exists() {
		try {
			URL url = getURL();
			if (ResourceUtils.isFileURL(url)) { 
				// Proceed with file system resolution
				return getFile().exists();
			}
			else {
				// Try a URL connection content-length header
				URLConnection con = url.openConnection();
				customizeConnection(con); //自定义Connection
				HttpURLConnection httpCon =
						(con instanceof HttpURLConnection ? (HttpURLConnection) con : null);
				if (httpCon != null) {
					int code = httpCon.getResponseCode(); //返回响应码
					if (code == HttpURLConnection.HTTP_OK) {
						return true;
					}
					else if (code == HttpURLConnection.HTTP_NOT_FOUND) {
						return false;
					}
				}
				if (con.getContentLengthLong() >= 0) { //响应内容长度
					return true;
				}
				if (httpCon != null) {
					// No HTTP OK status, and no content-length header: give up
					httpCon.disconnect();
					return false;
				}
				else {
					// Fall back to stream existence: can we open the stream?
					getInputStream().close();  //使用输入流是否可关闭判断
					return true;
				}
			}
		}
		catch (IOException ex) {
			return false;
		}
	}

	@Override
	public boolean isReadable() {
		try {
			URL url = getURL();
			if (ResourceUtils.isFileURL(url)) {  //File是否可读且不为目录
				// Proceed with file system resolution
				File file = getFile();
				return (file.canRead() && !file.isDirectory());
			}
			else { //不为文件时直接返回True
				return true;
			}
		}
		catch (IOException ex) {
			return false;
		}
	}

	@Override
	public boolean isFile() {
		try {
			URL url = getURL();
			if (url.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) { //vfs协议
				return VfsResourceDelegate.getResource(url).isFile();
			}
			return ResourceUtils.URL_PROTOCOL_FILE.equals(url.getProtocol()); //file协议
		}
		catch (IOException ex) {
			return false;
		}
	}

	/**
	 * This implementation returns a File reference for the underlying class path 返回底层类路径资源的文件引用
	 * resource, provided that it refers to a file in the file system. 提供文件系统中
	 * @see org.springframework.util.ResourceUtils#getFile(java.net.URL, String)
	 */
	@Override
	public File getFile() throws IOException {
		URL url = getURL();
		if (url.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
			return VfsResourceDelegate.getResource(url).getFile();
		}
		return ResourceUtils.getFile(url, getDescription()); //从Url中直接获取File对象
	}

	/**
	 * This implementation determines the underlying File
	 * (or jar file, in case of a resource in a jar/zip).
	 */
	@Override
	protected File getFileForLastModifiedCheck() throws IOException {
		URL url = getURL();
		if (ResourceUtils.isJarURL(url)) { //判断是否为jar url
			URL actualUrl = ResourceUtils.extractArchiveURL(url); //Jar文件中的实际URL
			if (actualUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				return VfsResourceDelegate.getResource(actualUrl).getFile();
			}
			return ResourceUtils.getFile(actualUrl, "Jar URL");
		}
		else {
			return getFile();
		}
	}

	/**
	 * This implementation returns a File reference for the given URI-identified 从给定URI资源中返回文件引用
	 * resource, provided that it refers to a file in the file system.
	 * @since 5.0
	 * @see #getFile(URI)
	 */
	protected boolean isFile(URI uri) {
		try {
			if (uri.getScheme().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				return VfsResourceDelegate.getResource(uri).isFile();
			}
			return ResourceUtils.URL_PROTOCOL_FILE.equals(uri.getScheme());
		}
		catch (IOException ex) {
			return false;
		}
	}

	/**
	 * This implementation returns a File reference for the given URI-identified
	 * resource, provided that it refers to a file in the file system.
	 * @see org.springframework.util.ResourceUtils#getFile(java.net.URI, String)
	 */
	protected File getFile(URI uri) throws IOException {
		if (uri.getScheme().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
			return VfsResourceDelegate.getResource(uri).getFile();
		}
		return ResourceUtils.getFile(uri, getDescription());
	}

	/**
	 * This implementation returns a FileChannel for the given URI-identified 从给定的URI中返回一个File Channel管道
	 * resource, provided that it refers to a file in the file system.
	 * @since 5.0
	 * @see #getFile()
	 */
	@Override
	public ReadableByteChannel readableChannel() throws IOException {
		try {
			// Try file system channel
			return FileChannel.open(getFile().toPath(), StandardOpenOption.READ); 
		}
		catch (FileNotFoundException | NoSuchFileException ex) {
			// Fall back to InputStream adaptation in superclass
			return super.readableChannel(); //
		}
	}

	@Override
	public long contentLength() throws IOException {
		URL url = getURL();
		if (ResourceUtils.isFileURL(url)) {
			// Proceed with file system resolution
			return getFile().length(); //文件长度
		}
		else {
			// Try a URL connection content-length header
			URLConnection con = url.openConnection();
			customizeConnection(con);
			return con.getContentLengthLong(); //响应内容长度
		}
	}

	@Override
	public long lastModified() throws IOException {
		URL url = getURL();
		if (ResourceUtils.isFileURL(url) || ResourceUtils.isJarURL(url)) {
			// Proceed with file system resolution
			try {
				return super.lastModified();
			}
			catch (FileNotFoundException ex) {
				// Defensively fall back to URL connection check instead
			}
		}
		// Try a URL connection last-modified header
		URLConnection con = url.openConnection();
		customizeConnection(con);
		return con.getLastModified();
	}

	/**
	 * Customize the given {@link URLConnection}, obtained in the course of an //自定义给定的URLConnection,从exists、contentLength、lastModified调用获取
	 * {@link #exists()}, {@link #contentLength()} or {@link #lastModified()} call.
	 * <p>Calls {@link ResourceUtils#useCachesIfNecessary(URLConnection)} and //调用缓存的URLConnection
	 * delegates to {@link #customizeConnection(HttpURLConnection)} if possible.委托给customizeConnection
	 * Can be overridden in subclasses.
	 * @param con the URLConnection to customize
	 * @throws IOException if thrown from URLConnection methods
	 */
	protected void customizeConnection(URLConnection con) throws IOException {
		ResourceUtils.useCachesIfNecessary(con);
		if (con instanceof HttpURLConnection) {
			customizeConnection((HttpURLConnection) con);
		}
	}

	/**
	 * Customize the given {@link HttpURLConnection}, obtained in the course of an
	 * {@link #exists()}, {@link #contentLength()} or {@link #lastModified()} call.
	 * <p>Sets request method "HEAD" by default. Can be overridden in subclasses. //默认设置的方法为HEAD，能够被子类从新
	 * @param con the HttpURLConnection to customize
	 * @throws IOException if thrown from HttpURLConnection methods
	 */
	protected void customizeConnection(HttpURLConnection con) throws IOException {
		con.setRequestMethod("HEAD");
	}


	/**
	 * Inner delegate class, avoiding a hard JBoss VFS API dependency at runtime.
	 */
	private static class VfsResourceDelegate {

		public static Resource getResource(URL url) throws IOException {
			return new VfsResource(VfsUtils.getRoot(url));
		}

		public static Resource getResource(URI uri) throws IOException {
			return new VfsResource(VfsUtils.getRoot(uri));
		}
	}

}

```

### ClassPathResource:类路径资源，Resource接口的核心实现

![](/img/post.img/spring/ClassPathResource.png)

1.路径path 不以"/"开头，内部会删除
2.未提供类加载器时将使用默认的类加载器


```java


package org.springframework.core.io;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.URL;

import org.springframework.lang.Nullable;
import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
import org.springframework.util.ObjectUtils;
import org.springframework.util.StringUtils;

/**
 * {@link Resource} implementation for class path resources(类路径的资源). Uses either a
 * given {@link ClassLoader} or a given {@link Class} for loading resources. 使用一个给定的类加载器或者类来加载资源
 *
 * <p>Supports resolution as {@code java.io.File} if the class path
 * resource resides in the file system, but not for resources in a JAR.
 * Always supports resolution as URL.
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 28.12.2003
 * @see ClassLoader#getResourceAsStream(String)
 * @see Class#getResourceAsStream(String)
 */
public class ClassPathResource extends AbstractFileResolvingResource {

	private final String path; //路径

	@Nullable
	private ClassLoader classLoader; //类加载器

	@Nullable
	private Class<?> clazz; //类


	/**
	 * Create a new {@code ClassPathResource} for {@code ClassLoader} usage. 创建一个新的ClassPathResource，使用ClassLoader
	 * A leading slash will be removed(前"/"将删除), as the ClassLoader resource access 作为类加载器资源访问方法将不接受"/"开头
	 * methods will not accept it.
	 * <p>The thread context class loader will be used for
	 * loading the resource.
	 * @param path the absolute path within the class path //类路径中的绝对路径
	 * @see java.lang.ClassLoader#getResourceAsStream(String)
	 * @see org.springframework.util.ClassUtils#getDefaultClassLoader()
	 */
	public ClassPathResource(String path) {
		this(path, (ClassLoader) null);
	}

	/**
	 * Create a new {@code ClassPathResource} for {@code ClassLoader} usage.
	 * A leading slash will be removed, as the ClassLoader resource access
	 * methods will not accept it.
	 * @param path the absolute path within the classpath
	 * @param classLoader the class loader to load the resource with,
	 * or {@code null} for the thread context class loader
	 * @see ClassLoader#getResourceAsStream(String)
	 */
	public ClassPathResource(String path, @Nullable ClassLoader classLoader) {
		Assert.notNull(path, "Path must not be null");
		String pathToUse = StringUtils.cleanPath(path); //标准化路径
		if (pathToUse.startsWith("/")) { //以"/"开头时进行截取
			pathToUse = pathToUse.substring(1);
		}
		this.path = pathToUse;
		this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
	}

	/**
	 * Create a new {@code ClassPathResource} for {@code Class} usage.
	 * The path can be relative to the given class, or absolute within
	 * the classpath via a leading slash.
	 * @param path relative or absolute path within the class path
	 * @param clazz the class to load resources with
	 * @see java.lang.Class#getResourceAsStream
	 */
	public ClassPathResource(String path, @Nullable Class<?> clazz) {
		Assert.notNull(path, "Path must not be null");
		this.path = StringUtils.cleanPath(path);
		this.clazz = clazz;
	}

	/**
	 * Create a new {@code ClassPathResource} with optional {@code ClassLoader}
	 * and {@code Class}. Only for internal usage.
	 * @param path relative or absolute path within the classpath
	 * @param classLoader the class loader to load the resource with, if any
	 * @param clazz the class to load resources with, if any
	 * @deprecated as of 4.3.13, in favor of selective use of
	 * {@link #ClassPathResource(String, ClassLoader)} vs {@link #ClassPathResource(String, Class)}
	 */
	@Deprecated
	protected ClassPathResource(String path, @Nullable ClassLoader classLoader, @Nullable Class<?> clazz) {
		this.path = StringUtils.cleanPath(path);
		this.classLoader = classLoader;
		this.clazz = clazz;
	}


	/**
	 * Return the path for this resource(返回当前资源的路径) (as resource path within the class path 作为类路径中的资源路径).
	 */
	public final String getPath() {
		return this.path;
	}

	/**
	 * Return the ClassLoader that this resource will be obtained from.返回当前资源的累加器
	 */
	@Nullable
	public final ClassLoader getClassLoader() {
		return (this.clazz != null ? this.clazz.getClassLoader() : this.classLoader);
	}


	/**
	 * This implementation checks for the resolution of a resource URL.
	 * @see java.lang.ClassLoader#getResource(String)
	 * @see java.lang.Class#getResource(String)
	 */
	@Override
	public boolean exists() {
		return (resolveURL() != null);
	}

	/**
	 * Resolves a URL for the underlying class path resource.解析底层类路径资源为一个URL：分别使用class、构造方法传入或者默认的ClassLoader、JDK提供的ClassLoader、
	 * @return the resolved URL, or {@code null} if not resolvable 返回解析后的URL null为不能解析
	 */
	@Nullable
	protected URL resolveURL() {
		if (this.clazz != null) {
			return this.clazz.getResource(this.path);
		}
		else if (this.classLoader != null) {
			return this.classLoader.getResource(this.path);
		}
		else {
			return ClassLoader.getSystemResource(this.path);
		}
	}

	/**
	 * This implementation opens an InputStream for the given class path resource.给定类路径资源打开输入流
	 * @see java.lang.ClassLoader#getResourceAsStream(String)
	 * @see java.lang.Class#getResourceAsStream(String)
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		InputStream is;
		if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
		if (is == null) {
			throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
		}
		return is;
	}

	/**
	 * This implementation returns a URL for the underlying class path resource,
	 * if available.
	 * @see java.lang.ClassLoader#getResource(String)
	 * @see java.lang.Class#getResource(String)
	 */
	@Override
	public URL getURL() throws IOException {
		URL url = resolveURL();
		if (url == null) {
			throw new FileNotFoundException(getDescription() + " cannot be resolved to URL because it does not exist");
		}
		return url;
	}

	/**
	 * This implementation creates a ClassPathResource, applying the given path（应用给定的相对路径）
	 * relative to the path of the underlying resource of this descriptor.
	 * @see org.springframework.util.StringUtils#applyRelativePath(String, String)
	 */
	@Override
	public Resource createRelative(String relativePath) {
		String pathToUse = StringUtils.applyRelativePath(this.path, relativePath);
		return (this.clazz != null ? new ClassPathResource(pathToUse, this.clazz) :
				new ClassPathResource(pathToUse, this.classLoader));
	}

	/**
	 * This implementation returns the name of the file that this class path
	 * resource refers to.
	 * @see org.springframework.util.StringUtils#getFilename(String)
	 */
	@Override
	@Nullable
	public String getFilename() {
		return StringUtils.getFilename(this.path); 
	}

	/**
	 * This implementation returns a description that includes the class path location.
	 */
	@Override
	public String getDescription() {
		StringBuilder builder = new StringBuilder("class path resource [");
		String pathToUse = path;
		if (this.clazz != null && !pathToUse.startsWith("/")) {
			builder.append(ClassUtils.classPackageAsResourcePath(this.clazz));
			builder.append('/');
		}
		if (pathToUse.startsWith("/")) {
			pathToUse = pathToUse.substring(1);
		}
		builder.append(pathToUse);
		builder.append(']');
		return builder.toString();
	}


	/**
	 * This implementation compares the underlying class path locations.
	 */
	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof ClassPathResource)) {
			return false;
		}
		ClassPathResource otherRes = (ClassPathResource) other;
		return (this.path.equals(otherRes.path) &&
				ObjectUtils.nullSafeEquals(this.classLoader, otherRes.classLoader) &&
				ObjectUtils.nullSafeEquals(this.clazz, otherRes.clazz));
	}

	/**
	 * This implementation returns the hash code of the underlying
	 * class path location.
	 */
	@Override
	public int hashCode() {
		return this.path.hashCode();
	}

}

```

### ServletContextResource:Servlet上下文资源

除了继承AbstractFileResolvingResource外还实现了上下文资源ContextResource接口

```java

package org.springframework.web.context.support;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.MalformedURLException;
import java.net.URL;

import javax.servlet.ServletContext;

import org.springframework.core.io.AbstractFileResolvingResource;
import org.springframework.core.io.ContextResource;
import org.springframework.core.io.Resource;
import org.springframework.lang.Nullable;
import org.springframework.util.Assert;
import org.springframework.util.ResourceUtils;
import org.springframework.util.StringUtils;
import org.springframework.web.util.WebUtils;

/**
 * {@link org.springframework.core.io.Resource} implementation for
 * {@link javax.servlet.ServletContext} resources, interpreting
 * relative paths within the web application root directory.
 *
 * <p>Always supports stream access and URL access, but only allows
 * {@code java.io.File} access when the web application archive
 * is expanded.
 *
 * @author Juergen Hoeller
 * @since 28.12.2003
 * @see javax.servlet.ServletContext#getResourceAsStream
 * @see javax.servlet.ServletContext#getResource
 * @see javax.servlet.ServletContext#getRealPath
 */
public class ServletContextResource extends AbstractFileResolvingResource implements ContextResource {

	private final ServletContext servletContext;//所有的操作都以servletContext进行

	private final String path;


	/**
	 * Create a new ServletContextResource.
	 * <p>The Servlet spec requires that resource paths start with a slash, Servlet规范需要，资源路径以“/”开头
	 * even if many containers accept paths without leading slash too. 即使许多容器接收的路径也不以"/"开头
	 * Consequently, the given path will be prepended with a slash if it 因此，如果没有以"/"开头将"/"作为前缀
	 * doesn't already start with one.
	 * @param servletContext the ServletContext to load from 从ServletContext加载资源路径
	 * @param path the path of the resource
	 */
	public ServletContextResource(ServletContext servletContext, String path) {
		// check ServletContext
		Assert.notNull(servletContext, "Cannot resolve ServletContextResource without ServletContext");
		this.servletContext = servletContext;

		// check path
		Assert.notNull(path, "Path is required");
		String pathToUse = StringUtils.cleanPath(path); //标准化资源路径
		if (!pathToUse.startsWith("/")) { //不以"/"开头是添加
			pathToUse = "/" + pathToUse;
		}
		this.path = pathToUse;
	}


	/**
	 * Return the ServletContext for this resource.
	 */
	public final ServletContext getServletContext() {
		return this.servletContext;
	}

	/**
	 * Return the path for this resource.
	 */
	public final String getPath() {
		return this.path;
	}

	/**
	 * This implementation checks {@code ServletContext.getResource}.
	 * @see javax.servlet.ServletContext#getResource(String)
	 */
	@Override
	public boolean exists() {
		try {
			URL url = this.servletContext.getResource(this.path);
			return (url != null);
		}
		catch (MalformedURLException ex) {
			return false;
		}
	}

	/**
	 * This implementation delegates to {@code ServletContext.getResourceAsStream},
	 * which returns {@code null} in case of a non-readable resource (e.g. a directory).
	 * @see javax.servlet.ServletContext#getResourceAsStream(String)
	 */
	@Override
	public boolean isReadable() {
		InputStream is = this.servletContext.getResourceAsStream(this.path);
		if (is != null) {
			try {
				is.close();
			}
			catch (IOException ex) {
				// ignore
			}
			return true;
		}
		else {
			return false;
		}
	}

	@Override
	public boolean isFile() {
		try {
			URL url = this.servletContext.getResource(this.path);
			if (url != null && ResourceUtils.isFileURL(url)) {
				return true;
			}
			else {
				return (this.servletContext.getRealPath(this.path) != null);
			}
		}
		catch (MalformedURLException ex) {
			return false;
		}
	}

	/**
	 * This implementation delegates to {@code ServletContext.getResourceAsStream},
	 * but throws a FileNotFoundException if no resource found.
	 * @see javax.servlet.ServletContext#getResourceAsStream(String)
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		InputStream is = this.servletContext.getResourceAsStream(this.path);
		if (is == null) {
			throw new FileNotFoundException("Could not open " + getDescription());
		}
		return is;
	}

	/**
	 * This implementation delegates to {@code ServletContext.getResource},
	 * but throws a FileNotFoundException if no resource found.
	 * @see javax.servlet.ServletContext#getResource(String)
	 */
	@Override
	public URL getURL() throws IOException {
		URL url = this.servletContext.getResource(this.path);
		if (url == null) {
			throw new FileNotFoundException(
					getDescription() + " cannot be resolved to URL because it does not exist");
		}
		return url;
	}

	/**
	 * This implementation resolves "file:" URLs(解析URL file：协议) or alternatively delegates to 委托给ServletContext.getRealPath
	 * {@code ServletContext.getRealPath}, throwing a FileNotFoundException 
	 * if not found or not resolvable.
	 * @see javax.servlet.ServletContext#getResource(String)
	 * @see javax.servlet.ServletContext#getRealPath(String)
	 */
	@Override
	public File getFile() throws IOException {
		URL url = this.servletContext.getResource(this.path);
		if (url != null && ResourceUtils.isFileURL(url)) {  //为file:时直接使用AbstractFileResolvingResource.getFile();
			// Proceed with file system resolution...
			return super.getFile();
		}
		else {
			String realPath = WebUtils.getRealPath(this.servletContext, this.path); //获取servletContext、path的真实路径
			return new File(realPath);
		}
	}

	/**
	 * This implementation creates a ServletContextResource, applying the given path 创建ServletContextResource资源，应用给定的底层文件的相对路径，
	 * relative to the path of the underlying file of this resource descriptor.
	 * @see org.springframework.util.StringUtils#applyRelativePath(String, String)
	 */
	@Override
	public Resource createRelative(String relativePath) {
		String pathToUse = StringUtils.applyRelativePath(this.path, relativePath); //获取当前路径下的相对路径，
		return new ServletContextResource(this.servletContext, pathToUse);
	}

	/**
	 * This implementation returns the name of the file that this ServletContext 返回此ServletContext资源引用的文件的名称
	 * resource refers to.
	 * @see org.springframework.util.StringUtils#getFilename(String)
	 */
	@Override
	@Nullable
	public String getFilename() {
		return StringUtils.getFilename(this.path);
	}

	/**
	 * This implementation returns a description that includes the ServletContext
	 * resource location.
	 */
	@Override
	public String getDescription() {
		return "ServletContext resource [" + this.path + "]";
	}

	@Override
	public String getPathWithinContext() {
		return this.path;
	}


	/**
	 * This implementation compares the underlying ServletContext resource locations.
	 */
	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof ServletContextResource)) {
			return false;
		}
		ServletContextResource otherRes = (ServletContextResource) other;
		return (this.servletContext.equals(otherRes.servletContext) && this.path.equals(otherRes.path));
	}

	/**
	 * This implementation returns the hash code of the underlying
	 * ServletContext resource location.
	 */
	@Override
	public int hashCode() {
		return this.path.hashCode();
	}

}

```


###  UrlResource: URL资源

```java

/**
 * {@link Resource} implementation for {@code java.net.URL} locators. 通过URL 定位资源
 * Supports resolution as a {@code URL} and also as a {@code File} in 支持解析为url 和在"file:"协议下的文件
 * case of the {@code "file:"} protocol.
 *
 * @author Juergen Hoeller
 * @since 28.12.2003
 * @see java.net.URL
 */
public class UrlResource extends AbstractFileResolvingResource {

	/**
	 * Original URI, if available; used for URI and File access. 如果可获取，原始的URI,使用URI和file访问
	 */
	@Nullable
	private final URI uri;

	/**
	 * Original URL, used for actual access.原始的URl 
	 */
	private final URL url; 

	/**
	 * Cleaned URL (with normalized path), used for comparisons. 与归一化路径,用于比较
	 */
	private final URL cleanedUrl;


	/**
	 * Create a new {@code UrlResource} based on the given URI object. 基于给定的URI对象创建UrlResource
	 * @param uri a URI
	 * @throws MalformedURLException if the given URL path is not valid
	 * @since 2.5
	 */
	public UrlResource(URI uri) throws MalformedURLException {
		Assert.notNull(uri, "URI must not be null");
		this.uri = uri;
		this.url = uri.toURL();
		this.cleanedUrl = getCleanedUrl(this.url, uri.toString());
	}

	/**
	 * Create a new {@code UrlResource} based on the given URL object. 基于给定的URL对象创建UrlResource
	 * @param url a URL
	 */
	public UrlResource(URL url) {
		Assert.notNull(url, "URL must not be null");
		this.url = url;
		this.cleanedUrl = getCleanedUrl(this.url, url.toString());
		this.uri = null;
	}

	/**
	 * Create a new {@code UrlResource} based on a URL path. 基于给定的URL路径创建UrlResource
	 * <p>Note: The given path needs to be pre-encoded if necessary.
	 * @param path a URL path
	 * @throws MalformedURLException if the given URL path is not valid
	 * @see java.net.URL#URL(String)
	 */
	public UrlResource(String path) throws MalformedURLException {
		Assert.notNull(path, "Path must not be null");
		this.uri = null;
		this.url = new URL(path);
		this.cleanedUrl = getCleanedUrl(this.url, path);
	}

	/**
	 * Create a new {@code UrlResource} based on a URI specification. 基于指定的URI创建UrlResource
	 * <p>The given parts will automatically get encoded if necessary. 给定的路径将自动编码
	 * @param protocol the URL protocol to use (e.g. "jar" or "file" - without colon); 协议
	 * also known as "scheme"
	 * @param location the location (e.g. the file path within that protocol);协议中的文件路径
	 * also known as "scheme-specific part"
	 * @throws MalformedURLException if the given URL specification is not valid
	 * @see java.net.URI#URI(String, String, String)
	 */
	public UrlResource(String protocol, String location) throws MalformedURLException  {
		this(protocol, location, null);
	}

	/**
	 * Create a new {@code UrlResource} based on a URI specification.
	 * <p>The given parts will automatically get encoded if necessary.
	 * @param protocol the URL protocol to use (e.g. "jar" or "file" - without colon);
	 * also known as "scheme"
	 * @param location the location (e.g. the file path within that protocol);
	 * also known as "scheme-specific part"
	 * @param fragment the fragment within that location (e.g. anchor on an HTML page,
	 * as following after a "#" separator) URL中的#锚点，分隔符
	 * @throws MalformedURLException if the given URL specification is not valid
	 * @see java.net.URI#URI(String, String, String)
	 */
	public UrlResource(String protocol, String location, @Nullable String fragment) throws MalformedURLException  {
		try {
			this.uri = new URI(protocol, location, fragment);
			this.url = this.uri.toURL();
			this.cleanedUrl = getCleanedUrl(this.url, this.uri.toString());
		}
		catch (URISyntaxException ex) {
			MalformedURLException exToThrow = new MalformedURLException(ex.getMessage());
			exToThrow.initCause(ex);
			throw exToThrow;
		}
	}


	/**
	 * Determine a cleaned URL for the given original URL. 
	 * @param originalUrl the original URL 原始URL
	 * @param originalPath the original URL path y原始URL 路径
	 * @return the cleaned URL
	 * @see org.springframework.util.StringUtils#cleanPath
	 */
	private URL getCleanedUrl(URL originalUrl, String originalPath) {
		try {
			return new URL(StringUtils.cleanPath(originalPath)); 
		}
		catch (MalformedURLException ex) {
			// Cleaned URL path cannot be converted to URL 不能构建RUL是返回原始的URL
			// -> take original URL.
			return originalUrl;
		}
	}

	/**
	 * This implementation opens an InputStream for the given URL.
	 * <p>It sets the {@code useCaches} flag to {@code false},
	 * mainly to avoid jar file locking on Windows.
	 * @see java.net.URL#openConnection()
	 * @see java.net.URLConnection#setUseCaches(boolean)
	 * @see java.net.URLConnection#getInputStream()
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		URLConnection con = this.url.openConnection();
		ResourceUtils.useCachesIfNecessary(con);
		try {
			return con.getInputStream();
		}
		catch (IOException ex) {
			// Close the HTTP connection (if applicable).
			if (con instanceof HttpURLConnection) {
				((HttpURLConnection) con).disconnect();
			}
			throw ex;
		}
	}

	/**
	 * This implementation returns the underlying URL reference.
	 */
	@Override
	public URL getURL() {
		return this.url;
	}

	/**
	 * This implementation returns the underlying URI directly,
	 * if possible.
	 */
	@Override
	public URI getURI() throws IOException {
		if (this.uri != null) {
			return this.uri;
		}
		else {
			return super.getURI();
		}
	}

	@Override
	public boolean isFile() {
		if (this.uri != null) {
			return super.isFile(this.uri);
		}
		else {
			return super.isFile();
		}
	}

	/**
	 * This implementation returns a File reference for the underlying URL/URI,
	 * provided that it refers to a file in the file system.
	 * @see org.springframework.util.ResourceUtils#getFile(java.net.URL, String)
	 */
	@Override
	public File getFile() throws IOException {
		if (this.uri != null) {
			return super.getFile(this.uri);
		}
		else {
			return super.getFile();
		}
	}

	/**
	 * This implementation creates a {@code UrlResource}, applying the given path
	 * relative to the path of the underlying URL of this resource descriptor.
	 * @see java.net.URL#URL(java.net.URL, String)
	 */
	@Override
	public Resource createRelative(String relativePath) throws MalformedURLException {
		if (relativePath.startsWith("/")) {
			relativePath = relativePath.substring(1);
		}
		return new UrlResource(new URL(this.url, relativePath));
	}

	/**
	 * This implementation returns the name of the file that this URL refers to.
	 * @see java.net.URL#getPath()
	 */
	@Override
	public String getFilename() {
		return StringUtils.getFilename(this.cleanedUrl.getPath());
	}

	/**
	 * This implementation returns a description that includes the URL.
	 */
	@Override
	public String getDescription() {
		return "URL [" + this.url + "]";
	}


	/**
	 * This implementation compares the underlying URL references.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof UrlResource &&
				this.cleanedUrl.equals(((UrlResource) other).cleanedUrl)));
	}

	/**
	 * This implementation returns the hash code of the underlying URL reference.
	 */
	@Override
	public int hashCode() {
		return this.cleanedUrl.hashCode();
	}

}


```

### FileUrlResource:文件URL资源，Spring 5.x提供，同时也实现了WritableResource接口(支持资源的可写入)

```java
package org.springframework.core.io;

import java.io.File;
import java.io.IOException;
import java.io.OutputStream;
import java.net.MalformedURLException;
import java.net.URL;
import java.nio.channels.FileChannel;
import java.nio.channels.WritableByteChannel;
import java.nio.file.Files;
import java.nio.file.StandardOpenOption;

import org.springframework.lang.Nullable;
import org.springframework.util.ResourceUtils;

/**
 * Subclass of {@link UrlResource} which assumes file resolution(UrlResource子类，假定文件解析), to the degree
 * of implementing the {@link WritableResource} interface for it. This resource
 * variant also caches resolved {@link File} handles from {@link #getFile()}. 此资源变体可缓存已解析的文件句柄
 *
 * <p>This is the class resolved by {@link DefaultResourceLoader} for a "file:..." 这是DefaultResourceLoader默认资源加载器URL位置为[file:...]的解析类
 * URL location, allowing a downcast to {@link WritableResource} for it. 运行向下转型为WritableResource
 *
 * <p>Alternatively, for direct construction from a {@link java.io.File} handle,
 * consider using {@link FileSystemResource}. For an NIO {@link java.nio.file.Path},
 * consider using {@link PathResource} instead.
 *
 * @author Juergen Hoeller
 * @since 5.0.2
 */
public class FileUrlResource extends UrlResource implements WritableResource {

	@Nullable
	private volatile File file; //使用volatile声明file


	/**
	 * Create a new {@code FileUrlResource} based on the given URL object. 基于Url创建FileUrlResource
	 * <p>Note that this does not enforce "file" as URL protocol. If a protocol
	 * is known to be resolvable to a file,
	 * @param url a URL
	 * @see ResourceUtils#isFileURL(URL)
	 * @see #getFile()
	 */
	public FileUrlResource(URL url) {
		super(url);
	}

	/**
	 * Create a new {@code FileUrlResource} based on the given file location,基于文件路径创建FileUrlResource
	 * using the URL protocol "file". 使用URL协议为file:
	 * <p>The given parts will automatically get encoded if necessary.给定的路径经自动编码
	 * @param location the location (i.e. the file path within that protocol)
	 * @throws MalformedURLException if the given URL specification is not valid
	 * @see UrlResource#UrlResource(String, String)
	 * @see ResourceUtils#URL_PROTOCOL_FILE
	 */
	public FileUrlResource(String location) throws MalformedURLException {
		super(ResourceUtils.URL_PROTOCOL_FILE, location);
	}


	@Override
	public File getFile() throws IOException {
		File file = this.file;
		if (file != null) {
			return file;
		}
		file = super.getFile();
		this.file = file;
		return file;
	}

	@Override
	public boolean isWritable() {
		try {
			URL url = getURL();
			if (ResourceUtils.isFileURL(url)) { //如果为文件路径
				// Proceed with file system resolution
				File file = getFile();
				return (file.canWrite() && !file.isDirectory());
			}
			else {
				return true;
			}
		}
		catch (IOException ex) {
			return false;
		}
	}

	@Override
	public OutputStream getOutputStream() throws IOException { //输出流
		return Files.newOutputStream(getFile().toPath()); 
	}

	@Override
	public WritableByteChannel writableChannel() throws IOException {
		return FileChannel.open(getFile().toPath(), StandardOpenOption.WRITE); //可写操作，使用时会覆盖以前的内容
	}

	@Override
	public Resource createRelative(String relativePath) throws MalformedURLException { //创建相对资源
		if (relativePath.startsWith("/")) {
			relativePath = relativePath.substring(1);
		}
		return new FileUrlResource(new URL(getURL(), relativePath));
	}

}

```


### BeanDefinitionResource: Bean定义资源，将Spring IOC中的Bean抽象为一个资源，内部持有BeanDefinition属性

```java

package org.springframework.beans.factory.support;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.core.io.AbstractResource;
import org.springframework.util.Assert;

/**
 * Descriptive {@link org.springframework.core.io.Resource} wrapper for 将BeanDefinition包装为一个描述性资源
 * a {@link org.springframework.beans.factory.config.BeanDefinition}.
 *
 * @author Juergen Hoeller
 * @since 2.5.2
 * @see org.springframework.core.io.DescriptiveResource
 */
class BeanDefinitionResource extends AbstractResource {

	private final BeanDefinition beanDefinition;


	/**
	 * Create a new BeanDefinitionResource. 
	 * @param beanDefinition the BeanDefinition objectto wrap 要包装的BeanDefinition对象
	 */
	public BeanDefinitionResource(BeanDefinition beanDefinition) {
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
		this.beanDefinition = beanDefinition;
	}

	/**
	 * Return the wrapped BeanDefinition object. 返回包装的BeanDefinition对象
	 */
	public final BeanDefinition getBeanDefinition() {
		return this.beanDefinition;
	}


	@Override
	public boolean exists() {
		return false;
	}

	@Override
	public boolean isReadable() {
		return false;
	}

	@Override
	public InputStream getInputStream() throws IOException {
		throw new FileNotFoundException(
				"Resource cannot be opened because it points to " + getDescription());
	}

	@Override
	public String getDescription() {
		return "BeanDefinition defined in " + this.beanDefinition.getResourceDescription();
	}


	/**
	 * This implementation compares the underlying BeanDefinition.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof BeanDefinitionResource &&
				((BeanDefinitionResource) other).beanDefinition.equals(this.beanDefinition)));
	}

	/**
	 * This implementation returns the hash code of the underlying BeanDefinition.
	 */
	@Override
	public int hashCode() {
		return this.beanDefinition.hashCode();
	}

}

```

### ByteArrayResource：字节数组资源，内部持有byteArray引用

```java

package org.springframework.core.io;

/**
 * {@link Resource} implementation for a given byte array. 通过字节数组实现Resource
 * <p>Creates a {@link ByteArrayInputStream} for the given byte array.
 *
 * <p>Useful for loading content from any given byte array, 从任何给定的字节数组加载内容非常有用
 * without having to resort to a single-use(一次性的) {@link InputStreamResource}.
 * Particularly useful for creating mail attachments from local content,对于从本地内容创建mail附件特别有用
 * where JavaMail needs to be able to read the stream multiple times.JavaMail需要能够多次读取流
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 1.2.3
 * @see java.io.ByteArrayInputStream
 * @see InputStreamResource
 * @see org.springframework.mail.javamail.MimeMessageHelper#addAttachment(String, InputStreamSource)
 */
public class ByteArrayResource extends AbstractResource {

	private final byte[] byteArray; //字节数组

	private final String description;


	/**
	 * Create a new {@code ByteArrayResource}.
	 * @param byteArray the byte array to wrap
	 */
	public ByteArrayResource(byte[] byteArray) {
		this(byteArray, "resource loaded from byte array");
	}

	/**
	 * Create a new {@code ByteArrayResource} with a description.
	 * @param byteArray the byte array to wrap
	 * @param description where the byte array comes from
	 */
	public ByteArrayResource(byte[] byteArray, @Nullable String description) {
		Assert.notNull(byteArray, "Byte array must not be null");
		this.byteArray = byteArray;
		this.description = (description != null ? description : "");
	}


	/**
	 * Return the underlying byte array.
	 */
	public final byte[] getByteArray() {
		return this.byteArray;
	}

	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean exists() {
		return true;
	}

	/**
	 * This implementation returns the length of the underlying byte array.
	 */
	@Override
	public long contentLength() {
		return this.byteArray.length;
	}

	/**
	 * This implementation returns a ByteArrayInputStream for the 返回字底层节数组的ByteArrayInputStream
	 * underlying byte array.
	 * @see java.io.ByteArrayInputStream
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		return new ByteArrayInputStream(this.byteArray);
	}

	/**
	 * This implementation returns a description that includes the passed-in
	 * {@code description}, if any.
	 */
	@Override
	public String getDescription() {
		return "Byte array resource [" + this.description + "]";
	}


	/**
	 * This implementation compares the underlying byte array.
	 * @see java.util.Arrays#equals(byte[], byte[])
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof ByteArrayResource &&
				Arrays.equals(((ByteArrayResource) other).byteArray, this.byteArray)));
	}

	/**
	 * This implementation returns the hash code based on the
	 * underlying byte array.
	 */
	@Override
	public int hashCode() {
		return (byte[].class.hashCode() * 29 * this.byteArray.length);
	}

}


```

### TransformedResource:转换资源，继承ByteArrayResource，可用于表示保存除内容外的所有其他信息的原始资源

```java
/**
 * An extension of {@link ByteArrayResource} that a {@link ResourceTransformer} ResourceTransformer扩展ByteArrayResource
 * can use to represent an original resource preserving all other information
 * except the content.  。
 *
 * @author Jeremy Grelle
 * @author Rossen Stoyanchev
 * @since 4.1
 */
public class TransformedResource extends ByteArrayResource {

	@Nullable
	private final String filename;//原始资源文件名称

	private final long lastModified;//原始资源修改时间


	public TransformedResource(Resource original, byte[] transformedContent) {
		super(transformedContent);
		this.filename = original.getFilename(); 
		try {
			this.lastModified = original.lastModified();
		}
		catch (IOException ex) {
			// should never happen
			throw new IllegalArgumentException(ex);
		}
	}


	@Override
	@Nullable
	public String getFilename() {
		return this.filename;
	}

	@Override
	public long lastModified() throws IOException {
		return this.lastModified;
	}

}

```


- DescriptiveResource:描述性资源

```java
package org.springframework.core.io;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;

import org.springframework.lang.Nullable;

/**
 * Simple {@link Resource} implementation that holds a resource description 简单实现，持有一个资源描述，但不指向实际可读的资源
 * but does not point to an actually readable resource.
 *
 * <p>To be used as placeholder if a {@code Resource} argument is ，如果一个资源参数是API期望，但不一定用于实际阅读时，DescriptiveResource将作为占位符使用
 * expected by an API but not necessarily used for actual reading.
 *
 * @author Juergen Hoeller
 * @since 1.2.6
 */
public class DescriptiveResource extends AbstractResource {

	private final String description;


	/**
	 * Create a new DescriptiveResource.
	 * @param description the resource description
	 */
	public DescriptiveResource(@Nullable String description) {
		this.description = (description != null ? description : "");
	}


	@Override
	public boolean exists() {
		return false;
	}

	@Override
	public boolean isReadable() {
		return false;
	}

	@Override
	public InputStream getInputStream() throws IOException {
		throw new FileNotFoundException(
				getDescription() + " cannot be opened because it does not point to a readable resource");
	}

	@Override
	public String getDescription() {
		return this.description;
	}


	/**
	 * This implementation compares the underlying description String.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof DescriptiveResource &&
				((DescriptiveResource) other).description.equals(this.description)));
	}

	/**
	 * This implementation returns the hash code of the underlying description String.
	 */
	@Override
	public int hashCode() {
		return this.description.hashCode();
	}

}


```

### FileSystemResource：文件系统资源，支出可写操作(WritableResource)


```java
/**
 * {@link Resource} implementation for {@code java.io.File} handles. 用于File文件处理
 * Supports resolution as a {@code File} and also as a {@code URL}. 支出解析为File和URL
 * Implements the extended {@link WritableResource} interface.扩展WritableResource接口
 *
 * <p>Note: As of Spring Framework 5.0, this {@link Resource} implementation 从Spring5.x开始使用NIO.2 API 读写交互
 * uses NIO.2 API for read/write interactions. Nevertheless(然而), in contrast to 与PathResource形成对比
 * {@link PathResource}, it primarily manages a {@code java.io.File} handle.它主要管理文件句柄
 *
 * @author Juergen Hoeller
 * @since 28.12.2003
 * @see PathResource
 * @see java.io.File
 * @see java.nio.file.Files
 */
public class FileSystemResource extends AbstractResource implements WritableResource {

	private final File file; //文件

	private final String path; //路径


	/**
	 * Create a new {@code FileSystemResource} from a {@link File} handle.从文件句柄中创建新的FileSystemResource
	 * <p>Note: When building relative resources via {@link #createRelative},当使用createRelative创建相对资源时，
	 * the relative path will apply <i>at the same directory level</i>:相对路径将使用相同的目录级别，
	 * e.g. new File("C:/dir1"), relative path "dir2" -> "C:/dir2"!
	 * If you prefer to have relative paths(喜欢使用相对路径) built underneath the given root(在给定的根目录下创建)
	 * directory, use the {@link #FileSystemResource(String) constructor with a file path}通过文件路径使用FileSystemResource(String)构造方法
	 * to append a trailing slash to the root path(在给路径后添加"/"): "C:/dir1/", which
	 * indicates this directory as root for all relative paths.指示此目录为所有相对路径的根目录
	 * @param file a File handle
	 */
	public FileSystemResource(File file) {
		Assert.notNull(file, "File must not be null");
		this.file = file;
		this.path = StringUtils.cleanPath(file.getPath());
	}

	/**
	 * Create a new {@code FileSystemResource} from a file path.从文件路径创建FileSystemResource
	 * <p>Note: When building relative resources via {@link #createRelative},
	 * it makes a difference whether the specified resource base path here
	 * ends with a slash or not(指定资源的基础目录以"/"结尾或者不是). In the case of "C:/dir1/", relative paths
	 * will be built underneath that root: e.g. relative path "dir2" ->
	 * "C:/dir1/dir2". In the case of "C:/dir1", relative paths will apply 相对路径将使用相同的目录级别
	 * at the same directory level: relative path "dir2" -> "C:/dir2".
	 * @param path a file path
	 */
	public FileSystemResource(String path) {
		Assert.notNull(path, "Path must not be null");
		this.file = new File(path);
		this.path = StringUtils.cleanPath(path);
	}


	/**
	 * Return the file path for this resource.
	 */
	public final String getPath() {
		return this.path;
	}

	/**
	 * This implementation returns whether the underlying file exists.
	 * @see java.io.File#exists()
	 */
	@Override
	public boolean exists() {
		return this.file.exists();
	}

	/**
	 * This implementation checks whether the underlying file is marked as readable
	 * (and corresponds to an actual file with content, not to a directory).不是目录
	 * @see java.io.File#canRead()
	 * @see java.io.File#isDirectory()
	 */
	@Override
	public boolean isReadable() {
		return (this.file.canRead() && !this.file.isDirectory());
	}

	/**
	 * This implementation opens a NIO file stream for the underlying file.为底层文件打开NIO文件流
	 * @see java.io.FileInputStream
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		try {
			return Files.newInputStream(this.file.toPath());
		}
		catch (NoSuchFileException ex) {
			throw new FileNotFoundException(ex.getMessage());
		}
	}

	/**
	 * This implementation checks whether the underlying file is marked as writable
	 * (and corresponds to an actual file with content, not to a directory).
	 * @see java.io.File#canWrite()
	 * @see java.io.File#isDirectory()
	 */
	@Override
	public boolean isWritable() {
		return (this.file.canWrite() && !this.file.isDirectory());// 可写操作，且不为目录
	}

	/**
	 * This implementation opens a FileOutputStream for the underlying file.
	 * @see java.io.FileOutputStream
	 */
	@Override
	public OutputStream getOutputStream() throws IOException {
		return Files.newOutputStream(this.file.toPath());
	}

	/**
	 * This implementation returns a URL for the underlying file. 返回文件的URL
	 * @see java.io.File#toURI()
	 */
	@Override
	public URL getURL() throws IOException {
		return this.file.toURI().toURL(); 
	}

	/**
	 * This implementation returns a URI for the underlying file. 返回文件的URI
	 * @see java.io.File#toURI()
	 */
	@Override
	public URI getURI() throws IOException {
		return this.file.toURI();
	}

	/**
	 * This implementation always indicates a file.
	 */
	@Override
	public boolean isFile() {
		return true;
	}

	/**
	 * This implementation returns the underlying File reference.
	 */
	@Override
	public File getFile() {
		return this.file;
	}

	/**
	 * This implementation opens a FileChannel for the underlying file.
	 * @see java.nio.channels.FileChannel
	 */
	@Override
	public ReadableByteChannel readableChannel() throws IOException {
		try {
			return FileChannel.open(this.file.toPath(), StandardOpenOption.READ);
		}
		catch (NoSuchFileException ex) {
			throw new FileNotFoundException(ex.getMessage());
		}
	}

	/**
	 * This implementation opens a FileChannel for the underlying file.
	 * @see java.nio.channels.FileChannel
	 */
	@Override
	public WritableByteChannel writableChannel() throws IOException {
		return FileChannel.open(this.file.toPath(), StandardOpenOption.WRITE);
	}

	/**
	 * This implementation returns the underlying File's length.
	 */
	@Override
	public long contentLength() throws IOException {
		return this.file.length();
	}

	/**
	 * This implementation creates a FileSystemResource, applying the given path
	 * relative to the path of the underlying file of this resource descriptor.
	 * @see org.springframework.util.StringUtils#applyRelativePath(String, String)
	 */
	@Override
	public Resource createRelative(String relativePath) {
		String pathToUse = StringUtils.applyRelativePath(this.path, relativePath);
		return new FileSystemResource(pathToUse);
	}

	/**
	 * This implementation returns the name of the file.
	 * @see java.io.File#getName()
	 */
	@Override
	public String getFilename() {
		return this.file.getName();
	}

	/**
	 * This implementation returns a description that includes the absolute
	 * path of the file.
	 * @see java.io.File#getAbsolutePath()
	 */
	@Override
	public String getDescription() {
		return "file [" + this.file.getAbsolutePath() + "]";
	}


	/**
	 * This implementation compares the underlying File references.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof FileSystemResource &&
				this.path.equals(((FileSystemResource) other).path)));
	}

	/**
	 * This implementation returns the hash code of the underlying File reference.
	 */
	@Override
	public int hashCode() {
		return this.path.hashCode();
	}

}

```

### InputStreamResource：输入流资源

```java
/**
 * {@link Resource} implementation for a given {@link InputStream}. 对于给定的InputStream
 * <p>Should only be used if no other specific {@code Resource} implementation 仅在没有其他特定Resource实现适用的情况下使用
 * is applicable. In particular(尤其), prefer {@link ByteArrayResource} or any of the 喜欢ByteArrayResource或者任何基于文件资源的实现
 * file-based {@code Resource} implementations where possible.
 *
 * <p>In contrast to other {@code Resource} implementations(与其他Resource接口实现), this is a descriptor InputStreamResource 描述一个已经打开的资源，
 * for an <i>already opened</i> resource - therefore returning {@code true} from 因此isOpen返回true
 * {@link #isOpen()}. Do not use an {@code InputStreamResource} if you need to 如果需要保存资源描述到其他地方或者需要从一个流中读取，不要使用InputStreamResource
 * keep the resource descriptor somewhere, or if you need to read from a stream
 * multiple times.
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 28.12.2003
 * @see ByteArrayResource
 * @see ClassPathResource
 * @see FileSystemResource
 * @see UrlResource
 */
public class InputStreamResource extends AbstractResource {

	private final InputStream inputStream;

	private final String description;

	private boolean read = false;


	/**
	 * Create a new InputStreamResource.
	 * @param inputStream the InputStream to use
	 */
	public InputStreamResource(InputStream inputStream) {
		this(inputStream, "resource loaded through InputStream");
	}

	/**
	 * Create a new InputStreamResource.
	 * @param inputStream the InputStream to use
	 * @param description where the InputStream comes from
	 */
	public InputStreamResource(InputStream inputStream, @Nullable String description) {
		Assert.notNull(inputStream, "InputStream must not be null");
		this.inputStream = inputStream;
		this.description = (description != null ? description : "");
	}


	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean exists() {
		return true;
	}

	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean isOpen() {
		return true;
	}

	/**
	 * This implementation throws IllegalStateException if attempting to
	 * read the underlying stream multiple times. 不支持多次读取
	 */
	@Override
	public InputStream getInputStream() throws IOException, IllegalStateException {
		if (this.read) {
			throw new IllegalStateException("InputStream has already been read - " +
					"do not use InputStreamResource if a stream needs to be read multiple times");
		}
		this.read = true;
		return this.inputStream;
	}

	/**
	 * This implementation returns a description that includes the passed-in
	 * description, if any.
	 */
	@Override
	public String getDescription() {
		return "InputStream resource [" + this.description + "]";
	}


	/**
	 * This implementation compares the underlying InputStream.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof InputStreamResource &&
				((InputStreamResource) other).inputStream.equals(this.inputStream)));
	}

	/**
	 * This implementation returns the hash code of the underlying InputStream.
	 */
	@Override
	public int hashCode() {
		return this.inputStream.hashCode();
	}

}


```

### PathResource：路径资源。Spring4.x提供，支持可写(WritableResource)

```java
/**
 * {@link Resource} implementation for {@code java.nio.file.Path} handles. 对于Java7中Path提供的实现 
 * Supports resolution as File, and also as URL.支持解析为File或者URL
 * Implements the extended {@link WritableResource} interface.
 *
 * @author Philippe Marschall
 * @author Juergen Hoeller
 * @since 4.0
 * @see java.nio.file.Path
 * @see java.nio.file.Files
 * @see FileSystemResource
 */
public class PathResource extends AbstractResource implements WritableResource {

	private final Path path; //Path引用


	/**
	 * Create a new PathResource from a Path handle. 从Path创建PathResource
	 * <p>Note: Unlike {@link FileSystemResource}, when building relative resources 当创建相对资源时，
	 * via {@link #createRelative}, the relative path will be built <i>underneath</i> 相对资源将在根路径的下面
	 * the given root: e.g. Paths.get("C:/dir1/"), relative path "dir2" -> "C:/dir1/dir2"! 
	 * @param path a Path handle
	 */
	public PathResource(Path path) {
		Assert.notNull(path, "Path must not be null");
		this.path = path.normalize(); //标准化
	}

	/**
	 * Create a new PathResource from a Path handle. 路径
	 * <p>Note: Unlike {@link FileSystemResource}, when building relative resources
	 * via {@link #createRelative}, the relative path will be built <i>underneath</i>
	 * the given root: e.g. Paths.get("C:/dir1/"), relative path "dir2" -> "C:/dir1/dir2"!
	 * @param path a path
	 * @see java.nio.file.Paths#get(String, String...)
	 */
	public PathResource(String path) {
		Assert.notNull(path, "Path must not be null");
		this.path = Paths.get(path).normalize();
	}

	/**
	 * Create a new PathResource from a Path handle.
	 * <p>Note: Unlike {@link FileSystemResource}, when building relative resources
	 * via {@link #createRelative}, the relative path will be built <i>underneath</i>
	 * the given root: e.g. Paths.get("C:/dir1/"), relative path "dir2" -> "C:/dir1/dir2"!
	 * @param uri a path URI
	 * @see java.nio.file.Paths#get(URI)
	 */
	public PathResource(URI uri) {
		Assert.notNull(uri, "URI must not be null");
		this.path = Paths.get(uri).normalize();
	}


	/**
	 * Return the file path for this resource. 返回路径
	 */
	public final String getPath() {
		return this.path.toString();
	}

	/**
	 * This implementation returns whether the underlying file exists. 文件是否存在
	 * @see java.nio.file.Files#exists(Path, java.nio.file.LinkOption...)
	 */
	@Override
	public boolean exists() {
		return Files.exists(this.path);
	}

	/**
	 * This implementation checks whether the underlying file is marked as readable
	 * (and corresponds to an actual file with content, not to a directory).
	 * @see java.nio.file.Files#isReadable(Path)
	 * @see java.nio.file.Files#isDirectory(Path, java.nio.file.LinkOption...)
	 */
	@Override
	public boolean isReadable() {
		return (Files.isReadable(this.path) && !Files.isDirectory(this.path));
	}

	/**
	 * This implementation opens a InputStream for the underlying file.
	 * @see java.nio.file.spi.FileSystemProvider#newInputStream(Path, OpenOption...)
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		if (!exists()) { //文件不存在
			throw new FileNotFoundException(getPath() + " (no such file or directory)");
		}
		if (Files.isDirectory(this.path)) { //文件未目录
			throw new FileNotFoundException(getPath() + " (is a directory)");
		}
		return Files.newInputStream(this.path);
	}

	/**
	 * This implementation checks whether the underlying file is marked as writable
	 * (and corresponds to an actual file with content, not to a directory).
	 * @see java.nio.file.Files#isWritable(Path)
	 * @see java.nio.file.Files#isDirectory(Path, java.nio.file.LinkOption...)
	 */
	@Override
	public boolean isWritable() {
		return (Files.isWritable(this.path) && !Files.isDirectory(this.path));
	}

	/**
	 * This implementation opens a OutputStream for the underlying file.
	 * @see java.nio.file.spi.FileSystemProvider#newOutputStream(Path, OpenOption...)
	 */
	@Override
	public OutputStream getOutputStream() throws IOException {
		if (Files.isDirectory(this.path)) {
			throw new FileNotFoundException(getPath() + " (is a directory)");
		}
		return Files.newOutputStream(this.path);
	}

	/**
	 * This implementation returns a URL for the underlying file.
	 * @see java.nio.file.Path#toUri()
	 * @see java.net.URI#toURL()
	 */
	@Override
	public URL getURL() throws IOException {
		return this.path.toUri().toURL();
	}

	/**
	 * This implementation returns a URI for the underlying file.
	 * @see java.nio.file.Path#toUri()
	 */
	@Override
	public URI getURI() throws IOException {
		return this.path.toUri();
	}

	/**
	 * This implementation always indicates a file.
	 */
	@Override
	public boolean isFile() {
		return true;
	}

	/**
	 * This implementation returns the underlying File reference.
	 */
	@Override
	public File getFile() throws IOException {
		try {
			return this.path.toFile(); //toFile
		}
		catch (UnsupportedOperationException ex) {
			// Only paths on the default file system can be converted to a File:
			// Do exception translation for cases where conversion is not possible.
			throw new FileNotFoundException(this.path + " cannot be resolved to absolute file path");
		}
	}

	/**
	 * This implementation opens a Channel for the underlying file. 从文件中打开一个可读的字节管道
	 * @see Files#newByteChannel(Path, OpenOption...)
	 */
	@Override
	public ReadableByteChannel readableChannel() throws IOException {
		try {
			return Files.newByteChannel(this.path, StandardOpenOption.READ);
		}
		catch (NoSuchFileException ex) {
			throw new FileNotFoundException(ex.getMessage());
		}
	}

	/**
	 * This implementation opens a Channel for the underlying file. 从文件中打开一个写的字节管道
	 * @see Files#newByteChannel(Path, OpenOption...)
	 */
	@Override
	public WritableByteChannel writableChannel() throws IOException { 
		return Files.newByteChannel(this.path, StandardOpenOption.WRITE);
	}

	/**
	 * This implementation returns the underlying file's length.
	 */
	@Override
	public long contentLength() throws IOException {
		return Files.size(this.path);
	}

	/**
	 * This implementation returns the underlying File's timestamp.
	 * @see java.nio.file.Files#getLastModifiedTime(Path, java.nio.file.LinkOption...)
	 */
	@Override
	public long lastModified() throws IOException {
		// We can not use the superclass method since it uses conversion to a File(因为它使用到文件的转换) and
		// only a Path on the default file system can be converted to a File... Path在默认的文件系统中能够转换为一个File
		return Files.getLastModifiedTime(this.path).toMillis();
	}

	/**
	 * This implementation creates a PathResource, applying the given path 创建相对路径的PathResource
	 * relative to the path of the underlying file of this resource descriptor.
	 * @see java.nio.file.Path#resolve(String)
	 */
	@Override
	public Resource createRelative(String relativePath) throws IOException {
		return new PathResource(this.path.resolve(relativePath));
	}

	/**
	 * This implementation returns the name of the file.
	 * @see java.nio.file.Path#getFileName()
	 */
	@Override
	public String getFilename() {
		return this.path.getFileName().toString(); //文件名
	}

	@Override
	public String getDescription() {
		return "path [" + this.path.toAbsolutePath() + "]";
	}


	/**
	 * This implementation compares the underlying Path references.
	 */
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof PathResource &&
				this.path.equals(((PathResource) other).path)));
	}

	/**
	 * This implementation returns the hash code of the underlying Path reference.
	 */
	@Override
	public int hashCode() {
		return this.path.hashCode();
	}

}
```
