---
layout: post
title:  "Spring源码-PathMatcher路径匹配器"
date:   2019-04-07 00:56:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}


PathMatcher为策略接口，用于基于字符串的路径匹配，Spring提供的默认实现为AntPathMatcher(Ant-style path patterns)




### PathMatcher：接口定义

```java
/**
 * Strategy interface for {@code String}-based path matching. 基于String路径匹配的策略接口
 *
 * <p>Used by {@link org.springframework.core.io.support.PathMatchingResourcePatternResolver}, 使用在PathMatchingResourcePatternResolver(路径匹配资源模式解析器)AbstractUrlHandlerMapping(抽象URL处理器映射器)、WebContentInterceptor(wen内容拦截器)
 * {@link org.springframework.web.servlet.handler.AbstractUrlHandlerMapping},
 * and {@link org.springframework.web.servlet.mvc.WebContentInterceptor}.
 *
 * <p>The default implementation is {@link AntPathMatcher}, supporting the 默认实现为AntPathMatcher
 * Ant-style pattern syntax.支持Ant-style模式语法
 *
 * @author Juergen Hoeller
 * @since 1.2
 * @see AntPathMatcher
 */
public interface PathMatcher {

	/**
	 * Does the given {@code path} represent a pattern that can be matched
	 * by an implementation of this interface? 给定的路径代表一个模式，能够匹配接口的实现
	 * <p>If the return value is {@code false}, then the {@link #match} 返回false时，match方法不需要使用，因为相等比较静态路径字符串将导致相同的接口，也就是
	 * method does not have to be used because direct equality comparisons
	 * on the static path Strings will lead to the same result.
	 * @param path the path String to check 要检查的路径字符串
	 * @return {@code true} if the given {@code path} represents a pattern 如果给定的路径代表一个模式返回true
	 */
	boolean isPattern(String path);

	/**
	 * Match the given {@code path} against the given {@code pattern}, 根据给定的模式匹配给定的路径
	 * according to this PathMatcher's matching strategy. 根据这个路径匹配器的匹配策略
	 * @param pattern the pattern to match against 要匹配的模式
	 * @param path the path String to test 要测试的路径字符串
	 * @return {@code true} if the supplied {@code path} matched,如果提供的路径匹配，返回true
	 * {@code false} if it didn't
	 */
	boolean match(String pattern, String path);

	/**
	 * Match the given {@code path} against the corresponding part of(对应的部分) the given 根据给定对应的部分模式匹配给定的路径
	 * {@code pattern}, according to this PathMatcher's matching strategy. 
	 * <p>Determines whether the pattern at least matches as far as the given base 确定模式至少匹配给定的基本路径
	 * path goes, assuming that a full path may then match as well.假设一个完整的路径也可能匹配
	 * @param pattern the pattern to match against
	 * @param path the path String to test
	 * @return {@code true} if the supplied {@code path} matched,
	 * {@code false} if it didn't
	 */
	boolean matchStart(String pattern, String path);

	/**
	 * Given a pattern and a full path(给定一个模式和完整的路径), determine the pattern-mapped part. 确认模式映射的部分
	 * <p>This method is supposed(假设) to find out which part of the path is matched 找出部分路径的动态匹配，通过真实的路径
	 * dynamically through an actual pattern, that is, it strips off(剥掉) a statically 它从给定的完整路径中删除静态定义的引导路径
	 * defined leading path from the given full path, returning only the actually 返回只有真实模式匹配部分的路径
	 * pattern-matched part of the path.
	 * <p>For example: For "myroot/*.html" as pattern and "myroot/myfile.html" 
	 * as full path, this method should return "myfile.html". The detailed 
	 * determination rules are specified to this PathMatcher's matching strategy. 详细的确定规则指定给这个路径匹配器的匹配策略
	 * <p>A simple implementation may return the given full path as-is(照原来的样子) in case 简单的实现可能返回给定的全路径,在实际模式的情况下
	 * of an actual pattern, and the empty String in case of the pattern not 空字符串的情况下模式不包含任何动态的部分
	 * containing any dynamic parts (i.e. the {@code pattern} parameter being 模式参数作为不符合实际模式的静态路径
	 * a static path that wouldn't qualify as an actual {@link #isPattern pattern}).
	 * A sophisticated implementation will differentiate between the static parts 复杂实现与静态部分和给定路径模式的动态部分不同
	 * and the dynamic parts of the given path pattern.
	 * @param pattern the path pattern 路径木事
	 * @param path the full path to introspect 完整自省路径
	 * @return the pattern-mapped part of the given {@code path} 给定路径的模式映射部分
	 * (never {@code null})
	 */
	String extractPathWithinPattern(String pattern, String path);

	/**
	 * Given a pattern and a full path, extract the URI template variables 提取URI模板变量. URI template
	 * variables are expressed through curly brackets ('{' and '}'). URI模板变量通过"{}""表达
	 * <p>For example: For pattern "/hotels/{hotel}" and path "/hotels/1", this method will
	 * return a map containing "hotel"->"1". 返回模板变量key-> value 
	 * @param pattern the path pattern, possibly containing URI templates
	 * @param path the full path to extract template variables from
	 * @return a map, containing variable names as keys; variables values as values 包含的变量名作为key，变量值作为value
	 */
	Map<String, String> extractUriTemplateVariables(String pattern, String path);

	/**
	 * Given a full path, returns a {@link Comparator} suitable for sorting patterns 返回适当的比较器用于排序模式，为了明确这条路径
	 * in order of explicitness for that path.
	 * <p>The full algorithm used depends on the underlying implementation, 完整的算法依赖于接口底层的实现
	 * but generally, the returned {@code Comparator} will 通常，返回
	 * {@linkplain java.util.List#sort(java.util.Comparator) sort}
	 * a list so that more specific patterns come before generic patterns. 个列表，以便更具体的模式出现在泛型模式之前
	 * @param path the full path to use for comparison 全路径用于比较
	 * @return a comparator capable of sorting patterns 排查模式比较的能力 in order of explicitness(为了表达清楚)
	 */
	Comparator<String> getPatternComparator(String path);

	/**
	 * Combines two patterns into a new pattern that is returned. 组合两个模式作为一个新的模式返回
	 * <p>The full algorithm used for combining the two pattern depends on the underlying implementation.
	 * @param pattern1 the first pattern
	 * @param pattern2 the second pattern
	 * @return the combination of the two patterns
	 * @throws IllegalArgumentException when the two patterns cannot be combined
	 */
	String combine(String pattern1, String pattern2);

}

```

## AntPathMatcher：Ant风格路径匹配器


### 示例

```java
PathMatcher pathMatcher=new AntPathMatcher();

Assert.assertTrue( pathMatcher.match("/static/**","/static/js/jquery.js"));

Assert.assertTrue(pathMatcher.match("/static/*.html","/static/index.html"));

Assert.assertTrue(pathMatcher.match("{config:[a-z]+.yml}","config.yml"));

Map<String, String> configUriTemplateVariables = pathMatcher.extractUriTemplateVariables("config/**/{config:[a-z]+.yml}", "config/config.yml");

Assert.assertEquals("config.yml",configUriTemplateVariables.get("config"));


Assert.assertTrue(pathMatcher.match("/api/person/{personId}","/api/person/1"));


Map<String, String> apiUriTemplateVariables = pathMatcher.extractUriTemplateVariables("/api/person/{personId}","/api/person/1");

Assert.assertEquals("1",apiUriTemplateVariables.get("personId"));
```


### 类结构
![](/img/post.img/spring/PathMatcher.png)

### 具体实现


#### 类描述：

```java
/**
 * {@link PathMatcher} implementation for Ant-style path patterns.PathMatcher接口实现，用于 Ant-style 路径模式匹配
 *
 * <p>Part of this mapping code has been kindly borrowed from <a href="http://ant.apache.org">Apache Ant</a>.
 *
 * <p>The mapping matches URLs using the following rules:<br> URL映射匹配规则
 * <ul>
 * <li>{@code ?} matches one character</li> 匹配一个字符
 * <li>{@code *} matches zero or more characters</li> 匹配0个或多个字符
 * <li>{@code **} matches zero or more <em>directories</em> in a path</li>在路径中 匹配0个或多个目录
 * <li>{@code {spring:[a-z]+}} matches the regexp {@code [a-z]+} as a path variable named "spring"</li> 匹配正则表达式作为路径参数名
 * </ul>
 *
 * <h3>Examples</h3>
 * <ul>
 * <li>{@code com/t?st.jsp} &mdash; matches {@code com/test.jsp} but also
 * {@code com/tast.jsp} or {@code com/txst.jsp}</li>
 * <li>{@code com/*.jsp} &mdash; matches all {@code .jsp} files in the
 * {@code com} directory</li>
 * <li><code>com/&#42;&#42;/test.jsp</code> &mdash; matches all {@code test.jsp}
 * files underneath the {@code com} path</li>
 * <li><code>org/springframework/&#42;&#42;/*.jsp</code> &mdash; matches all
 * {@code .jsp} files underneath the {@code org/springframework} path</li>
 * <li><code>org/&#42;&#42;/servlet/bla.jsp</code> &mdash; matches
 * {@code org/springframework/servlet/bla.jsp} but also
 * {@code org/springframework/testing/servlet/bla.jsp} and {@code org/servlet/bla.jsp}</li>
 * <li>{@code com/{filename:\\w+}.jsp} will match {@code com/test.jsp} and assign the value {@code test}
 * to the {@code filename} variable</li>
 * </ul>
 *
 * <p><strong>Note:</strong> a pattern and a path must both be absolute or must 模式和路径必须都是绝对路径或者必须都为相对路径
 * both be relative in order for the two to match(为了使两者匹配). Therefore it is recommended 因此推荐使用当前实现的用户以"/"作为前缀
 * that users of this implementation to sanitize patterns in order to prefix
 * them with "/" as it makes sense in the context in which they're used. 这在使用它们的上下文中是有意义的
 *
 */
public class AntPathMatcher implements PathMatcher {

}

```


#### 成员变量

```java
/** Default path separator: "/" */
public static final String DEFAULT_PATH_SEPARATOR = "/"; // 默认路径分隔符

private static final int CACHE_TURNOFF_THRESHOLD = 65536; //缓存阈值开关，当tokenizedPatternCache与stringMatcherCache的存储大于该值时将清空

private static final Pattern VARIABLE_PATTERN = Pattern.compile("\\{[^/]+?\\}"); //变量模式

private static final char[] WILDCARD_CHARS = { '*', '?', '{' }; //通配符


private String pathSeparator; //路径分割符

private PathSeparatorPatternCache pathSeparatorPatternCache; //路径分隔模式缓存

private boolean caseSensitive = true; //区分大小写，默认为true

private boolean trimTokens = false; //削减标记

@Nullable
private volatile Boolean cachePatterns; //使用volatile修饰的缓存模式，内存可见

private final Map<String, String[]> tokenizedPatternCache = new ConcurrentHashMap<>(256); //标记模式缓存，key为路径模式，value为标记的多个部分

final Map<String, AntPathStringMatcher> stringMatcherCache = new ConcurrentHashMap<>(256); //缓存ANT路径字符串匹配器, key的模式，value为AntPathStringMatcher

```

#### 构造方法

```java
/**
 * Create a new instance with the {@link #DEFAULT_PATH_SEPARATOR}. 使用默认的路径分隔符创建实例
 */
public AntPathMatcher() {
	this.pathSeparator = DEFAULT_PATH_SEPARATOR;
	this.pathSeparatorPatternCache = new PathSeparatorPatternCache(DEFAULT_PATH_SEPARATOR);
}

/**
 * A convenient, alternative constructor to use with a custom path separator.
 * @param pathSeparator the path separator to use, must not be {@code null}.
 * @since 4.1
 */
public AntPathMatcher(String pathSeparator) {
	Assert.notNull(pathSeparator, "'pathSeparator' is required");
	this.pathSeparator = pathSeparator;
	this.pathSeparatorPatternCache = new PathSeparatorPatternCache(pathSeparator);
}

```

#### 核心方法

```java
/**
 * Set the path separator to use for pattern parsing. 设置路径分割符用于模式解析
 * <p>Default is "/", as in Ant.
 */
public void setPathSeparator(@Nullable String pathSeparator) {
	this.pathSeparator = (pathSeparator != null ? pathSeparator : DEFAULT_PATH_SEPARATOR);
	this.pathSeparatorPatternCache = new PathSeparatorPatternCache(this.pathSeparator);
}

/**
 * Specify whether to perform pattern matching in a case-sensitive fashion. 指定是否以区分大小写的方式执行模式匹配
 * <p>Default is {@code true}. Switch this to {@code false} for case-insensitive(不区分大小写) matching.
 * @since 4.2
 */
public void setCaseSensitive(boolean caseSensitive) {
	this.caseSensitive = caseSensitive;
}

/**
 * Specify whether to trim tokenized paths and patterns. 指定是否trim标记的路径和模式
 * <p>Default is {@code false}. 默认false
 */
public void setTrimTokens(boolean trimTokens) {
	this.trimTokens = trimTokens;
}

/**
 * Specify whether to cache parsed pattern metadata for patterns passed 指定是否缓存解析模式的元数据，传递到match方法
 * into this matcher's {@link #match} method. A value of {@code true} true激活，无限模式缓存
 * activates an unlimited pattern cache; a value of {@code false} turns
 * the pattern cache off completely.
 * <p>Default is for the cache to be on, but with the variant to automatically 默认情况下，缓存是打开的
 * turn it off when encountering too many patterns to cache at runtime 当时遇到太多的模式在运行时缓存时，自动会关闭
 * (the threshold is 65536), assuming that arbitrary permutations of patterns 假设任意的排列模式进来，几乎不可能遇到重复出现的模式
 * are coming in, with little chance for encountering a recurring pattern.
 * @since 4.0.1
 * @see #getStringMatcher(String)
 */
public void setCachePatterns(boolean cachePatterns) {
	this.cachePatterns = cachePatterns;
}

private void deactivatePatternCache() { //禁用模式缓存
	this.cachePatterns = false;
	this.tokenizedPatternCache.clear();
	this.stringMatcherCache.clear();
}


@Override
public boolean isPattern(String path) { //是否匹配当前接口，路径中包含 "*"/"?"
	return (path.indexOf('*') != -1 || path.indexOf('?') != -1);
}

@Override
public boolean match(String pattern, String path) { //路径是否匹配模式
	return doMatch(pattern, path, true, null);
}

@Override
public boolean matchStart(String pattern, String path) {  //快速匹配
	return doMatch(pattern, path, false, null);
}

/**
 * Actually match the given {@code path} against the given {@code pattern}. 根据给定的模式匹配给定的路径
 * @param pattern the pattern to match against 要匹配的模式
 * @param path the path String to test 要测试的路径字符串
 * @param fullMatch whether a full pattern match is required (else a pattern match 是否全路径匹配，
 * as far as the given base path goes is sufficient)一个模式匹配快速的匹配给定的基本路径
 * @return {@code true} if the supplied {@code path} matched, {@code false} if it didn't
 */
protected boolean doMatch(String pattern, String path, boolean fullMatch,
		@Nullable Map<String, String> uriTemplateVariables) {

	if (path.startsWith(this.pathSeparator) != pattern.startsWith(this.pathSeparator)) { //路径与模式开头不一致直接返回false
		return false;
	}

	String[] pattDirs = tokenizePattern(pattern); //将给定的路径分词为多个部分 如"/static/*.js" 返回 {static,*.js}
	if (fullMatch && this.caseSensitive && !isPotentialMatch(path, pattDirs)) { //全路径、区分大小写、 是潜在的匹配
		return false;
	}

	String[] pathDirs = tokenizePath(path); //分词路径

	int pattIdxStart = 0;//模式id起始
	int pattIdxEnd = pattDirs.length - 1; //模式id结尾
	int pathIdxStart = 0;//路径id起始
	int pathIdxEnd = pathDirs.length - 1;//路径id结尾

	// Match all elements up to the first ** 将所有元素匹配到第一个"**"上 
	while (pattIdxStart <= pattIdxEnd && pathIdxStart <= pathIdxEnd) {
		String pattDir = pattDirs[pattIdxStart];
		if ("**".equals(pattDir)) {  //模式中遇到"**"直接跳出
			break;
		}
		if (!matchStrings(pattDir, pathDirs[pathIdxStart], uriTemplateVariables)) { //是否匹配字符串
			return false;
		}
		pattIdxStart++;
		pathIdxStart++;
	}
	// 当上面代码没有执行break时,也就意味着模式中不存在"**"匹配多个路径，pattIdxStart将>pattIdxEnd pathIdxStart> pathIdxEnd 
	//如："/static/*.html", "/static/index.html" 匹配时
	if (pathIdxStart > pathIdxEnd) { 
		// Path is exhausted, only match if rest of pattern is * or **'s 路径用完,只匹配"*"或者"**"模式
		if (pattIdxStart > pattIdxEnd) { // 模式id起始>模式id结尾
			return (pattern.endsWith(this.pathSeparator) == path.endsWith(this.pathSeparator)); //是否以相同的分隔符结尾
		}
		if (!fullMatch) { //不是全路径匹配直接返回true
			return true;
		}
		//pattIdxStart == pattIdxEnd 且后缀为* 、路径以分隔符结束时返回true
		if (pattIdxStart == pattIdxEnd && pattDirs[pattIdxStart].equals("*") && path.endsWith(this.pathSeparator)) {
			return true;
		}
		//pattIdxStart < pattIdxEnd 遍历所有的模式，如果都不为** 返回false
		for (int i = pattIdxStart; i <= pattIdxEnd; i++) {
			if (!pattDirs[i].equals("**")) {
				return false;
			}
		}
		return true;
	}
	else if (pattIdxStart > pattIdxEnd) {
		// String not exhausted, but pattern is. Failure. 模式匹配耗尽，肯定不匹配，直接返回false
		return false;
	}
	else if (!fullMatch && "**".equals(pattDirs[pattIdxStart])) { //不是全路径匹配且模式为**
		// Path start definitely matches due to "**" part in pattern. 由于模式中的“**”部分，路径开始绝对匹配
		return true;
	}

	// up to last '**' 如果模式中有"**"将执行下面循环，从后往往前遍历，遇到"**"跳出循环，
	while (pattIdxStart <= pattIdxEnd && pathIdxStart <= pathIdxEnd) {
		String pattDir = pattDirs[pattIdxEnd];
		if (pattDir.equals("**")) { 
			break;
		}
		if (!matchStrings(pattDir, pathDirs[pathIdxEnd], uriTemplateVariables)) {
			return false;
		}
		pattIdxEnd--;
		pathIdxEnd--;
	}
	if (pathIdxStart > pathIdxEnd) { //路径耗尽
		// String is exhausted
		for (int i = pattIdxStart; i <= pattIdxEnd; i++) {
			if (!pattDirs[i].equals("**")) {
				return false;
			}
		}
		return true;
	}
	//当再次遇到"**"时，且路径没有耗尽执行
	while (pattIdxStart != pattIdxEnd && pathIdxStart <= pathIdxEnd) {
		int patIdxTmp = -1;
		for (int i = pattIdxStart + 1; i <= pattIdxEnd; i++) {
			if (pattDirs[i].equals("**")) {
				patIdxTmp = i;
				break;
			}
		}
		if (patIdxTmp == pattIdxStart + 1) {
			// '**/**' situation, so skip one
			pattIdxStart++;
			continue;
		}
		// Find the pattern between padIdxStart & padIdxTmp in str between
		// strIdxStart & strIdxEnd
		int patLength = (patIdxTmp - pattIdxStart - 1);
		int strLength = (pathIdxEnd - pathIdxStart + 1);
		int foundIdx = -1;

		strLoop:
		for (int i = 0; i <= strLength - patLength; i++) {
			for (int j = 0; j < patLength; j++) {
				String subPat = pattDirs[pattIdxStart + j + 1];
				String subStr = pathDirs[pathIdxStart + i + j];
				if (!matchStrings(subPat, subStr, uriTemplateVariables)) {
					continue strLoop;
				}
			}
			foundIdx = pathIdxStart + i;
			break;
		}

		if (foundIdx == -1) {
			return false;
		}

		pattIdxStart = patIdxTmp;
		pathIdxStart = foundIdx + patLength;
	}

	for (int i = pattIdxStart; i <= pattIdxEnd; i++) {
		if (!pattDirs[i].equals("**")) {
			return false;
		}
	}

	return true;
}

private boolean isPotentialMatch(String path, String[] pattDirs) { //是否潜在匹配
	if (!this.trimTokens) { //不trim分词时
		int pos = 0;
		for (String pattDir : pattDirs) {
			int skipped = skipSeparator(path, pos, this.pathSeparator);
			pos += skipped;
			skipped = skipSegment(path, pos, pattDir);
			if (skipped < pattDir.length()) {
				return (skipped > 0 || (pattDir.length() > 0 && isWildcardChar(pattDir.charAt(0))));
			}
			pos += skipped;
		}
	}
	return true;
}

private int skipSegment(String path, int pos, String prefix) { //忽略部分
	int skipped = 0;
	for (int i = 0; i < prefix.length(); i++) {
		char c = prefix.charAt(i);
		if (isWildcardChar(c)) {
			return skipped;
		}
		int currPos = pos + skipped;
		if (currPos >= path.length()) {
			return 0;
		}
		if (c == path.charAt(currPos)) {
			skipped++;
		}
	}
	return skipped;
}

private int skipSeparator(String path, int pos, String separator) { //跳过分隔符，获取path的
	int skipped = 0;
	while (path.startsWith(separator, pos + skipped)) {
		skipped += separator.length();
	}
	return skipped;
}

//是否为通配符
private boolean isWildcardChar(char c) {
	for (char candidate : WILDCARD_CHARS) {
		if (c == candidate) {
			return true;
		}
	}
	return false;
}

/**
 * Tokenize the given path pattern into parts 分词路径模式为多个部分, based on this matcher's settings.基于此匹配器的设置
 * <p>Performs caching based on {@link #setCachePatterns}, 基于setCachePatterns执行缓存 delegating to 委托给tokenizePath用于真实的分词算法
 * {@link #tokenizePath(String)} for the actual tokenization algorithm.
 * @param pattern the pattern to tokenize 要分词的模式
 * @return the tokenized pattern parts 
 */
protected String[] tokenizePattern(String pattern) {
	String[] tokenized = null;
	Boolean cachePatterns = this.cachePatterns;
	if (cachePatterns == null || cachePatterns.booleanValue()) {
		tokenized = this.tokenizedPatternCache.get(pattern);
	}
	if (tokenized == null) {
		tokenized = tokenizePath(pattern);
		if (cachePatterns == null && this.tokenizedPatternCache.size() >= CACHE_TURNOFF_THRESHOLD) {
			// Try to adapt to the runtime situation that we're encountering:尝试适应我们所遇到的运行时情况
			// There are obviously too many different patterns coming in here... 这里显然有太多不同的模式
			// So let's turn off the cache since the patterns are unlikely to be reoccurring. 因此，让我们关闭缓存，因为模式不太可能再次发生
			deactivatePatternCache();
			return tokenized;
		}
		if (cachePatterns == null || cachePatterns.booleanValue()) { //缓存匹配的模式
			this.tokenizedPatternCache.put(pattern, tokenized);
		}
	}
	return tokenized;
}

/**
 * Tokenize the given path String into parts, based on this matcher's settings.于此匹配器的设置，分词给定的路径
 * @param path the path to tokenize
 * @return the tokenized path parts
 */
protected String[] tokenizePath(String path) {
	return StringUtils.tokenizeToStringArray(path, this.pathSeparator, this.trimTokens, true);
}

/**
 * Test whether or not a string matches against a pattern. 字符串与模式是否匹配
 * @param pattern the pattern to match against (never {@code null})
 * @param str the String which must be matched against the pattern (never {@code null})
 * @return {@code true} if the string matches against the pattern, or {@code false} otherwise
 */
private boolean matchStrings(String pattern, String str,
		@Nullable Map<String, String> uriTemplateVariables) {

	return getStringMatcher(pattern).matchStrings(str, uriTemplateVariables);
}

/**
 * Build or retrieve an {@link AntPathStringMatcher} for the given pattern. 对于给定的模式构建或者检索一个AntPathStringMatcher
 * <p>The default implementation checks this AntPathMatcher's internal cache 默认的实现检查当前AntPathMatcher内部缓存
 * (see {@link #setCachePatterns}), creating a new AntPathStringMatcher instance 创建一个新的AntPathStringMatcher实例
 * if no cached copy is found. 如果没有缓存
 * <p>When encountering too many patterns to cache at runtime (the threshold is 65536), 当遇到太多模式无法在运行时缓存时
 * it turns the default cache off 它关闭默认缓存, assuming that arbitrary permutations of patterns 假设模式的任意排列出现
 * are coming in, with little chance for encountering a recurring pattern.  几乎不可能遇到重复出现的模式
 * <p>This method may be overridden to implement a custom cache strategy. 可以重写此方法来实现自定义缓存策略
 * @param pattern the pattern to match against (never {@code null})
 * @return a corresponding AntPathStringMatcher (never {@code null})
 * @see #setCachePatterns
 */
protected AntPathStringMatcher getStringMatcher(String pattern) {
	AntPathStringMatcher matcher = null;
	Boolean cachePatterns = this.cachePatterns;
	if (cachePatterns == null || cachePatterns.booleanValue()) {
		matcher = this.stringMatcherCache.get(pattern); //从缓存中获取
	}
	if (matcher == null) {
		matcher = new AntPathStringMatcher(pattern, this.caseSensitive); //创建一个缓存
		if (cachePatterns == null && this.stringMatcherCache.size() >= CACHE_TURNOFF_THRESHOLD) {
			// Try to adapt to the runtime situation that we're encountering:
			// There are obviously too many different patterns coming in here...
			// So let's turn off the cache since the patterns are unlikely to be reoccurring.
			deactivatePatternCache();
			return matcher;
		}
		if (cachePatterns == null || cachePatterns.booleanValue()) {
			this.stringMatcherCache.put(pattern, matcher);
		}
	}
	return matcher;
}

/**
 * Given a pattern and a full path,给定一个模式和全路径 determine the pattern-mapped part. <p>For example: <ul> 确认模式映射的部分,也就是说返回模式匹配的部分路径
 * <li>'{@code /docs/cvs/commit.html}' and '{@code /docs/cvs/commit.html} -> ''</li> 
 * <li>'{@code /docs/*}' and '{@code /docs/cvs/commit} -> '{@code cvs/commit}'</li> 
 * <li>'{@code /docs/cvs/*.html}' and '{@code /docs/cvs/commit.html} -> '{@code commit.html}'</li>
 * <li>'{@code /docs/**}' and '{@code /docs/cvs/commit} -> '{@code cvs/commit}'</li>
 * <li>'{@code /docs/**\/*.html}' and '{@code /docs/cvs/commit.html} -> '{@code cvs/commit.html}'</li>
 * <li>'{@code /*.html}' and '{@code /docs/cvs/commit.html} -> '{@code docs/cvs/commit.html}'</li>
 * <li>'{@code *.html}' and '{@code /docs/cvs/commit.html} -> '{@code /docs/cvs/commit.html}'</li>
 * <li>'{@code *}' and '{@code /docs/cvs/commit.html} -> '{@code /docs/cvs/commit.html}'</li> </ul>
 * <p>Assumes that {@link #match} returns {@code true} for '{@code pattern}' and '{@code path}', but 假设match返回true，但是
 * does <strong>not</strong> enforce this. 执行当前方法不不强制执行
 */
@Override
public String extractPathWithinPattern(String pattern, String path) {
	String[] patternParts = StringUtils.tokenizeToStringArray(pattern, this.pathSeparator, this.trimTokens, true); //分词模式
	String[] pathParts = StringUtils.tokenizeToStringArray(path, this.pathSeparator, this.trimTokens, true); //分词路径
	StringBuilder builder = new StringBuilder(); //拼接返回的结果
	boolean pathStarted = false; //路径开始标志 "/"

	for (int segment = 0; segment < patternParts.length; segment++) {
		String patternPart = patternParts[segment];
		if (patternPart.indexOf('*') > -1 || patternPart.indexOf('?') > -1) { //分词模式中包含*或者？
			for (; segment < pathParts.length; segment++) {
				if (pathStarted || (segment == 0 && !pattern.startsWith(this.pathSeparator))) {
					builder.append(this.pathSeparator); //添加分隔符
				}
				builder.append(pathParts[segment]); //添加路径
				pathStarted = true;
			}
		}
	}

	return builder.toString();
}

@Override
public Map<String, String> extractUriTemplateVariables(String pattern, String path) {
	Map<String, String> variables = new LinkedHashMap<>();
	boolean result = doMatch(pattern, path, true, variables);
	if (!result) {
		throw new IllegalStateException("Pattern \"" + pattern + "\" is not a match for \"" + path + "\"");
	}
	return variables;
}

/**
 * Combine two patterns into a new pattern. 合并两个模式为一个新模式
 * <p>This implementation simply concatenates the two patterns, unless 当前实现简单的连接两个模式，除非第一个模式包含：文件扩展名匹配
 * the first pattern contains a file extension match (e.g., {@code *.html}).
 * In that case, the second pattern will be merged into the first. Otherwise, 在这种情况下，第二个模式经合并到第一个，否则抛出一次
 * an {@code IllegalArgumentException} will be thrown.
 * <h3>Examples</h3>
 * <table border="1">
 * <tr><th>Pattern 1</th><th>Pattern 2</th><th>Result</th></tr>
 * <tr><td>{@code null}</td><td>{@code null}</td><td>&nbsp;</td></tr>
 * <tr><td>/hotels</td><td>{@code null}</td><td>/hotels</td></tr>
 * <tr><td>{@code null}</td><td>/hotels</td><td>/hotels</td></tr>
 * <tr><td>/hotels</td><td>/bookings</td><td>/hotels/bookings</td></tr>
 * <tr><td>/hotels</td><td>bookings</td><td>/hotels/bookings</td></tr>
 * <tr><td>/hotels/*</td><td>/bookings</td><td>/hotels/bookings</td></tr>
 * <tr><td>/hotels/&#42;&#42;</td><td>/bookings</td><td>/hotels/&#42;&#42;/bookings</td></tr>
 * <tr><td>/hotels</td><td>{hotel}</td><td>/hotels/{hotel}</td></tr>
 * <tr><td>/hotels/*</td><td>{hotel}</td><td>/hotels/{hotel}</td></tr>
 * <tr><td>/hotels/&#42;&#42;</td><td>{hotel}</td><td>/hotels/&#42;&#42;/{hotel}</td></tr>
 * <tr><td>/*.html</td><td>/hotels.html</td><td>/hotels.html</td></tr>
 * <tr><td>/*.html</td><td>/hotels</td><td>/hotels.html</td></tr>
 * <tr><td>/*.html</td><td>/*.txt</td><td>{@code IllegalArgumentException}</td></tr>
 * </table>
 * @param pattern1 the first pattern
 * @param pattern2 the second pattern
 * @return the combination of the two patterns
 * @throws IllegalArgumentException if the two patterns cannot be combined
 */
@Override
public String combine(String pattern1, String pattern2) {
	if (!StringUtils.hasText(pattern1) && !StringUtils.hasText(pattern2)) {
		return ""; //都为空，返回""
	}
	if (!StringUtils.hasText(pattern1)) {
		return pattern2;
	}
	if (!StringUtils.hasText(pattern2)) {
		return pattern1;
	}

	boolean pattern1ContainsUriVar = (pattern1.indexOf('{') != -1); 包含"{"
	if (!pattern1.equals(pattern2) && !pattern1ContainsUriVar && match(pattern1, pattern2)) {
		// /* + /hotel -> /hotel ; "/*.*" + "/*.html" -> /*.html
		// However /user + /user -> /usr/user ; /{foo} + /bar -> /{foo}/bar
		return pattern2;
	}

	// /hotels/* + /booking -> /hotels/booking
	// /hotels/* + booking -> /hotels/booking
	if (pattern1.endsWith(this.pathSeparatorPatternCache.getEndsOnWildCard())) {
		return concat(pattern1.substring(0, pattern1.length() - 2), pattern2);
	}

	// /hotels/** + /booking -> /hotels/**/booking
	// /hotels/** + booking -> /hotels/**/booking
	if (pattern1.endsWith(this.pathSeparatorPatternCache.getEndsOnDoubleWildCard())) {
		return concat(pattern1, pattern2);
	}

	int starDotPos1 = pattern1.indexOf("*.");
	if (pattern1ContainsUriVar || starDotPos1 == -1 || this.pathSeparator.equals(".")) {
		// simply concatenate the two patterns
		return concat(pattern1, pattern2);
	}

	String ext1 = pattern1.substring(starDotPos1 + 1);
	int dotPos2 = pattern2.indexOf('.');
	String file2 = (dotPos2 == -1 ? pattern2 : pattern2.substring(0, dotPos2));
	String ext2 = (dotPos2 == -1 ? "" : pattern2.substring(dotPos2));
	boolean ext1All = (ext1.equals(".*") || ext1.equals(""));
	boolean ext2All = (ext2.equals(".*") || ext2.equals(""));
	if (!ext1All && !ext2All) {
		throw new IllegalArgumentException("Cannot combine patterns: " + pattern1 + " vs " + pattern2);
	}
	String ext = (ext1All ? ext2 : ext1);
	return file2 + ext;
}

private String concat(String path1, String path2) { //连接
	boolean path1EndsWithSeparator = path1.endsWith(this.pathSeparator);
	boolean path2StartsWithSeparator = path2.startsWith(this.pathSeparator);

	if (path1EndsWithSeparator && path2StartsWithSeparator) {
		return path1 + path2.substring(1);
	}
	else if (path1EndsWithSeparator || path2StartsWithSeparator) {
		return path1 + path2;
	}
	else {
		return path1 + this.pathSeparator + path2;
	}
}

/**
 * Given a full path 给定全路径, returns a {@link Comparator} suitable for sorting patterns  in order of
 * explicitness. 适当排序的模式明确的排序
 * <p>This{@code Comparator} will {@linkplain java.util.List#sort(Comparator) sort}
 * a list so that more specific patterns (without uri templates or wild cards) come before 具体的模式(不包含模板或者通配符)在泛型模式之前
 * generic patterns. So given a list with the following patterns:
 * <ol>
 * <li>{@code /hotels/new}</li>
 * <li>{@code /hotels/{hotel}}</li> <li>{@code /hotels/*}</li>
 * </ol>
 * the returned comparator will sort this list so that the order will be as indicated. 返回的比较器将排序这个集合，因此顺序就会显示出来
 * <p>The full path given as parameter is used to test for exact matches.：作为参数给出的完整路径用于测试精确匹配 So when the given path
 * is {@code /hotels/2}, the pattern {@code /hotels/2} will be sorted before {@code /hotels/1}.
 * @param path the full path to use for comparison
 * @return a comparator capable of sorting patterns in order of explicitness
 */
@Override
public Comparator<String> getPatternComparator(String path) {
	return new AntPatternComparator(path);
}


```

#### 内部类

```java
/**
 * Tests whether or not a string matches against a pattern via a {@link Pattern}. 测试字符串是否与模式匹配
 * <p>The pattern may contain special characters 模式可能包含指定的字符: 
 		'*' means zero or more characters; 匹配0个或者多个字符
 		'?' means one and only one character; 匹配一个且只有一个字符
 		'{' and '}' indicate a URI template pattern. For example <tt>/users/{user}</tt>."{}"表明URI模板匹配
 */
protected static class AntPathStringMatcher {
	//全局模式
	private static final Pattern GLOB_PATTERN = Pattern.compile("\\?|\\*|\\{((?:\\{[^/]+?\\}|[^/{}]|\\\\[{}])+?)\\}");

	private static final String DEFAULT_VARIABLE_PATTERN = "(.*)"; //默认参数模式

	private final Pattern pattern;

	private final List<String> variableNames = new LinkedList<>(); //参数名列表

	public AntPathStringMatcher(String pattern) {
		this(pattern, true);
	}
	//内部主要构造Pattern对象
	public AntPathStringMatcher(String pattern, boolean caseSensitive) { //区分大小写caseSensitive
		StringBuilder patternBuilder = new StringBuilder();
		Matcher matcher = GLOB_PATTERN.matcher(pattern); //全局模式匹配分词后的模式
		int end = 0;
		while (matcher.find()) { //查找
			patternBuilder.append(quote(pattern, end, matcher.start())); 
			String match = matcher.group(); //分组，转换成正则表达式相关的字符
			if ("?".equals(match)) {
				patternBuilder.append('.');
			}
			else if ("*".equals(match)) {
				patternBuilder.append(".*");
			}
			else if (match.startsWith("{") && match.endsWith("}")) {
				int colonIdx = match.indexOf(':'); //包含:
				if (colonIdx == -1) {
					patternBuilder.append(DEFAULT_VARIABLE_PATTERN); //添加默认参数匹配，
					this.variableNames.add(matcher.group(1)); //获取第一个分组的结果，即参数变量名
				}
				else { //进包括
					String variablePattern = match.substring(colonIdx + 1, match.length() - 1);
					patternBuilder.append('(');
					patternBuilder.append(variablePattern);
					patternBuilder.append(')');
					String variableName = match.substring(1, colonIdx);
					this.variableNames.add(variableName);
				}
			}
			end = matcher.end();
		}
		patternBuilder.append(quote(pattern, end, pattern.length()));
		this.pattern = (caseSensitive ? Pattern.compile(patternBuilder.toString()) :
				Pattern.compile(patternBuilder.toString(), Pattern.CASE_INSENSITIVE));
	}
	//返回文字模式：
	private String quote(String s, int start, int end) {
		if (start == end) {
			return "";
		}
		return Pattern.quote(s.substring(start, end));
	}

	/**
	 * Main entry point.
	 * @return {@code true} if the string matches against the pattern, or {@code false} otherwise. 如果字符串与模式匹配返回true
	 */
	public boolean matchStrings(String str, @Nullable Map<String, String> uriTemplateVariables) {
		Matcher matcher = this.pattern.matcher(str);
		if (matcher.matches()) { //匹配
			if (uriTemplateVariables != null) {
				// SPR-8455
				if (this.variableNames.size() != matcher.groupCount()) {
					throw new IllegalArgumentException("The number of capturing groups in the pattern segment " +
							this.pattern + " does not match the number of URI template variables it defines, " +
							"which can occur if capturing groups are used in a URI template regex. " +
							"Use non-capturing groups instead."); //uri模板中使用正则表达式
				}
				for (int i = 1; i <= matcher.groupCount(); i++) {
					String name = this.variableNames.get(i - 1); //获取参数变量
					String value = matcher.group(i); //分组结果
					uriTemplateVariables.put(name, value);
				}
			}
			return true;
		}
		else {
			return false;
		}
	}
}


/**
 * The default {@link Comparator} implementation returned by 默认比较器生成
 * {@link #getPatternComparator(String)}.
 * <p>In order, the most "generic" pattern is determined by the following: 
 * <ul>
 * <li>if it's null or a capture all pattern (i.e. it is equal to "/**")</li>
 * <li>if the other pattern is an actual match</li>
 * <li>if it's a catch-all pattern (i.e. it ends with "**"</li>
 * <li>if it's got more "*" than the other pattern</li>
 * <li>if it's got more "{foo}" than the other pattern</li>
 * <li>if it's shorter than the other pattern</li>
 * </ul>
 */
protected static class AntPatternComparator implements Comparator<String> {

	private final String path;

	public AntPatternComparator(String path) {
		this.path = path;
	}

	/**
	 * Compare two patterns to determine which should match first, i.e. which
	 * is the most specific regarding the current path.
	 * @return a negative integer, zero, or a positive integer as pattern1 is
	 * more specific, equally specific, or less specific than pattern2.
	 */
	@Override
	public int compare(String pattern1, String pattern2) {
		PatternInfo info1 = new PatternInfo(pattern1);
		PatternInfo info2 = new PatternInfo(pattern2);

		if (info1.isLeastSpecific() && info2.isLeastSpecific()) {
			return 0;
		}
		else if (info1.isLeastSpecific()) {
			return 1;
		}
		else if (info2.isLeastSpecific()) {
			return -1;
		}

		boolean pattern1EqualsPath = pattern1.equals(path);
		boolean pattern2EqualsPath = pattern2.equals(path);
		if (pattern1EqualsPath && pattern2EqualsPath) {
			return 0;
		}
		else if (pattern1EqualsPath) {
			return -1;
		}
		else if (pattern2EqualsPath) {
			return 1;
		}

		if (info1.isPrefixPattern() && info2.getDoubleWildcards() == 0) {
			return 1;
		}
		else if (info2.isPrefixPattern() && info1.getDoubleWildcards() == 0) {
			return -1;
		}

		if (info1.getTotalCount() != info2.getTotalCount()) {
			return info1.getTotalCount() - info2.getTotalCount();
		}

		if (info1.getLength() != info2.getLength()) {
			return info2.getLength() - info1.getLength();
		}

		if (info1.getSingleWildcards() < info2.getSingleWildcards()) {
			return -1;
		}
		else if (info2.getSingleWildcards() < info1.getSingleWildcards()) {
			return 1;
		}

		if (info1.getUriVars() < info2.getUriVars()) {
			return -1;
		}
		else if (info2.getUriVars() < info1.getUriVars()) {
			return 1;
		}

		return 0;
	}


	/**
	 * Value class that holds information about the pattern, e.g. number of 保存有关模式的信息
	 * occurrences of "*", "**", and "{" pattern elements.
	 */
	private static class PatternInfo {

		@Nullable
		private final String pattern; //要匹配的模式

		private int uriVars;

		private int singleWildcards; //一个通配符

		private int doubleWildcards; //两个通配符

		private boolean catchAllPattern; //匹配说有模式

		private boolean prefixPattern; //模式前缀

		@Nullable
		private Integer length;

		public PatternInfo(@Nullable String pattern) {
			this.pattern = pattern;
			if (this.pattern != null) {
				initCounters();
				this.catchAllPattern = this.pattern.equals("/**"); 
				this.prefixPattern = !this.catchAllPattern && this.pattern.endsWith("/**");
			}
			if (this.uriVars == 0) {
				this.length = (this.pattern != null ? this.pattern.length() : 0);
			}
		}
		//初始化成员变量
		protected void initCounters() {
			int pos = 0;
			if (this.pattern != null) {
				while (pos < this.pattern.length()) {
					if (this.pattern.charAt(pos) == '{') {
						this.uriVars++;
						pos++;
					}
					else if (this.pattern.charAt(pos) == '*') {
						if (pos + 1 < this.pattern.length() && this.pattern.charAt(pos + 1) == '*') {
							this.doubleWildcards++; 
							pos += 2;
						}
						else if (pos > 0 && !this.pattern.substring(pos - 1).equals(".*")) {
							this.singleWildcards++;
							pos++;
						}
						else {
							pos++;
						}
					}
					else {
						pos++;
					}
				}
			}
		}

		public int getUriVars() {
			return this.uriVars;
		}

		public int getSingleWildcards() {
			return this.singleWildcards;
		}

		public int getDoubleWildcards() {
			return this.doubleWildcards;
		}

		public boolean isLeastSpecific() {
			return (this.pattern == null || this.catchAllPattern);
		}

		public boolean isPrefixPattern() {
			return this.prefixPattern;
		}

		public int getTotalCount() {
			return this.uriVars + this.singleWildcards + (2 * this.doubleWildcards);
		}

		/**
		 * Returns the length of the given pattern 返回给定模式的长度, where template variables are considered to be 1 long. 模板变量长度为1
		 */
		public int getLength() {
			if (this.length == null) {
				this.length = (this.pattern != null ?
						VARIABLE_PATTERN.matcher(this.pattern).replaceAll("#").length() : 0);
			}
			return this.length;
		}
	}
}


/**
 * A simple cache for patterns that depend on the configured path separator.  一个简单的缓存，用于依赖于配置的路径分隔符的模式
 */
private static class PathSeparatorPatternCache {

	private final String endsOnWildCard;

	private final String endsOnDoubleWildCard;

	public PathSeparatorPatternCache(String pathSeparator) {
		this.endsOnWildCard = pathSeparator + "*";
		this.endsOnDoubleWildCard = pathSeparator + "**";
	}

	public String getEndsOnWildCard() {
		return this.endsOnWildCard;
	}

	public String getEndsOnDoubleWildCard() {
		return this.endsOnDoubleWildCard;
	}
}

```
