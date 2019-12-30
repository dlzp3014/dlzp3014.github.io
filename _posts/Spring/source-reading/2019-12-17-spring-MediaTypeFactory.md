---
layout: post
title:  "Spring源码-MediaTypeFactory媒体类型工厂"
date:   2019-12-17 08:38:00
categories: Spring 
tags: Spring-Source-Reading
---

* content
{:toc}

MediaTypeFactory用于从`Resource`资源或者文件名称解析MediaType对象，提供的静态方法如下:

- Optional<MediaType> getMediaType(@Nullable Resource resource)

从Resource对象中获取MediaType，如果没有找到返回NULL

```java
public static Optional<MediaType> getMediaType(@Nullable Resource resource) {
	return Optional.ofNullable(resource)
			.map(Resource::getFilename) //文件名
			.flatMap(MediaTypeFactory::getMediaType);//扁平流
}
```

- Optional<MediaType> getMediaType(@Nullable String filename)

从文件扩展名中获取MediaType，如果不存在返回NULL

```java
public static Optional<MediaType> getMediaType(@Nullable String filename) {
	return getMediaTypes(filename).stream().findFirst();//获取第一个
}
```

- List<MediaType> getMediaTypes(@Nullable String filename)

从文件扩展名中获取MediaType，如果不存在返回空列表


```java
public static List<MediaType> getMediaTypes(@Nullable String filename) {
	return Optional.ofNullable(StringUtils.getFilenameExtension(filename)) //获取文件名的扩展名
			.map(s -> s.toLowerCase(Locale.ENGLISH)) //映射：小写
			.map(fileExtensionToMediaTypes::get) //文件扩展名映射MediaType: 直接根据key获取
			.orElse(Collections.emptyList()); //不存在返回空集合
}
```

fileExtensionToMediaTypes: 文件扩展名映射媒体类型获取如下





```java
private static final String MIME_TYPES_FILE_NAME = "/org/springframework/http/mime.types"; //Spring框架提供的媒体类型路径

//key为扩展文件名，值为对应的媒体，
private static final MultiValueMap<String, MediaType> fileExtensionToMediaTypes = parseMimeTypes(); //解析


/**
 * Parse the {@code mime.types} file found in the resources. Format is:
 * <code>
 * # comments begin with a '#'<br>
 * # the format is &lt;mime type> &lt;space separated file extensions><br>
 * # for example:<br>
 * text/plain    txt text<br>
 * # this would map file.txt and file.text to<br>
 * # the mime type "text/plain"<br>
 * </code>
 * @return a multi-value map, mapping media types to file extensions.
 */
private static MultiValueMap<String, MediaType> parseMimeTypes() {
	//获取资源路径
	try (InputStream is = MediaTypeFactory.class.getResourceAsStream(MIME_TYPES_FILE_NAME)) {
		//缓冲区读取，编码为ASCII
		BufferedReader reader = new BufferedReader(new InputStreamReader(is, StandardCharsets.US_ASCII));
		MultiValueMap<String, MediaType> result = new LinkedMultiValueMap<>();
		String line;
		while ((line = reader.readLine()) != null) { //按行读取，忽略空行或者#开头(注释)
			if (line.isEmpty() || line.charAt(0) == '#') {
				continue;
			}
			//符号 行内容：text/plain					txt text conf def list log in
			String[] tokens = StringUtils.tokenizeToStringArray(line, " \t\n\r\f"); //【\t:水平制表符、\n:回车换行、\r:回车、\f:换页】
			MediaType mediaType = MediaType.parseMediaType(tokens[0]); //媒体类型:第一个分割后的数据，其他数据为扩展名
			for (int i = 1; i < tokens.length; i++) {
				String fileExtension = tokens[i].toLowerCase(Locale.ENGLISH); //小写
				result.add(fileExtension, mediaType);//添加
			}
		}
		return result;
	}
	catch (IOException ex) {
		throw new IllegalStateException("Could not load '" + MIME_TYPES_FILE_NAME + "'", ex);
	}
}
```


