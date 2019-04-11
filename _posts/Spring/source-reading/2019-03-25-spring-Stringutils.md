---
layout: post
title:  "Spring源码-StringUtils"
date:   2019-03-28 00:42:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}





## 常量：

```java
private static final String FOLDER_SEPARATOR = "/"; //文件夹分隔符

private static final String WINDOWS_FOLDER_SEPARATOR = "\\"; //windows文件夹分隔符

private static final String TOP_PATH = ".."; //顶层路径

private static final String CURRENT_PATH = "."; //当前路径

private static final char EXTENSION_SEPARATOR = '.'; //扩展分隔符
```

### 处理字符串方法：

```java
/**
 * Check whether the given {@code String} is empty. 检查给定的字符串是否为空
 * <p>This method accepts any Object as an argument, comparing it to(当前方法接收任何对象作为参数，比较null或者为空)
 * {@code null} and the empty String. As a consequence, this method
 * will never return {@code true} for a non-null non-String object.
 * <p>The Object signature is useful for general attribute handling code
 * that commonly deals with Strings but generally has to iterate over
 * Objects since attributes may e.g. be primitive value objects as well.
 * @param str the candidate String
 * @since 3.2.1
 */
public static boolean isEmpty(@Nullable Object str) {
	return (str == null || "".equals(str));
}

/**
 * Check that the given {@code CharSequence} is neither {@code null} nor 检查给定的字符序列有内容
 * of length 0.
 * <p>Note: this method returns {@code true} for a {@code CharSequence}
 * that purely consists of whitespace(空格).
 * <p><pre class="code">
 * StringUtils.hasLength(null) = false
 * StringUtils.hasLength("") = false
 * StringUtils.hasLength(" ") = true
 * StringUtils.hasLength("Hello") = true
 * </pre>
 * @param str the {@code CharSequence} to check (may be {@code null})
 * @return {@code true} if the {@code CharSequence} is not {@code null} and has length
 * @see #hasText(String)
 */
public static boolean hasLength(@Nullable CharSequence str) {
	return (str != null && str.length() > 0);
}

/**
 * Check that the given {@code String} is neither {@code null} nor of length 0.
 * <p>Note: this method returns {@code true} for a {@code String} that
 * purely consists of whitespace.
 * @param str the {@code String} to check (may be {@code null})
 * @return {@code true} if the {@code String} is not {@code null} and has length
 * @see #hasLength(CharSequence)
 * @see #hasText(String)
 */
public static boolean hasLength(@Nullable String str) {
	return (str != null && !str.isEmpty());
}

/**
 * Check whether the given {@code CharSequence} contains actual <em>text</em>. 文本
 * <p>More specifically, this method returns {@code true} if the
 * {@code CharSequence} is not {@code null}, its length is greater than
 * 0, and it contains at least one non-whitespace character.
 * <p><pre class="code">
 * StringUtils.hasText(null) = false
 * StringUtils.hasText("") = false
 * StringUtils.hasText(" ") = false
 * StringUtils.hasText("12345") = true
 * StringUtils.hasText(" 12345 ") = true
 * </pre>
 * @param str the {@code CharSequence} to check (may be {@code null})
 * @return {@code true} if the {@code CharSequence} is not {@code null},
 * its length is greater than 0, and it does not contain whitespace only
 * @see Character#isWhitespace
 */
public static boolean hasText(@Nullable CharSequence str) {
	return (str != null && str.length() > 0 && containsText(str));
}

/**
 * Check whether the given {@code String} contains actual <em>text</em>. 包含真实的文本
 * <p>More specifically, this method returns {@code true} if the 如果字符串不为null 长度大于0，且至少包含一个非空字符
 * {@code String} is not {@code null}, its length is greater than 0,
 * and it contains at least one non-whitespace character.
 * @param str the {@code String} to check (may be {@code null})
 * @return {@code true} if the {@code String} is not {@code null}, its
 * length is greater than 0, and it does not contain whitespace only
 * @see #hasText(CharSequence)
 */
public static boolean hasText(@Nullable String str) {
	return (str != null && !str.isEmpty() && containsText(str));
}

private static boolean containsText(CharSequence str) {
	int strLen = str.length();
	for (int i = 0; i < strLen; i++) {
		if (!Character.isWhitespace(str.charAt(i))) {
			return true;
		}
	}
	return false;
}

/**
 * Check whether the given {@code CharSequence} contains any whitespace characters. 包含任何空字符串
 * @param str the {@code CharSequence} to check (may be {@code null})
 * @return {@code true} if the {@code CharSequence} is not empty and
 * contains at least 1 whitespace character
 * @see Character#isWhitespace
 */
public static boolean containsWhitespace(@Nullable CharSequence str) {
	if (!hasLength(str)) {
		return false;
	}

	int strLen = str.length();
	for (int i = 0; i < strLen; i++) {
		if (Character.isWhitespace(str.charAt(i))) {
			return true;
		}
	}
	return false;
}

/**
 * Check whether the given {@code String} contains any whitespace characters.
 * @param str the {@code String} to check (may be {@code null})
 * @return {@code true} if the {@code String} is not empty and
 * contains at least 1 whitespace character
 * @see #containsWhitespace(CharSequence)
 */
public static boolean containsWhitespace(@Nullable String str) {
	return containsWhitespace((CharSequence) str);
}

/**
 * Trim leading and trailing whitespace from the given {@code String}.消减头和尾的空格：分别从前、从后进行
 * @param str the {@code String} to check
 * @return the trimmed {@code String}
 * @see java.lang.Character#isWhitespace
 */
public static String trimWhitespace(String str) {
	if (!hasLength(str)) {
		return str;
	}

	int beginIndex = 0;
	int endIndex = str.length() - 1;

	while (beginIndex <= endIndex && Character.isWhitespace(str.charAt(beginIndex))) {
		beginIndex++;
	}

	while (endIndex > beginIndex && Character.isWhitespace(str.charAt(endIndex))) {
		endIndex--;
	}

	return str.substring(beginIndex, endIndex + 1);
}

/**
 * Trim <i>all</i> whitespace from the given {@code String}: 消减全部的空格
 * leading, trailing, and in between characters.
 * @param str the {@code String} to check
 * @return the trimmed {@code String}
 * @see java.lang.Character#isWhitespace
 */
public static String trimAllWhitespace(String str) {
	if (!hasLength(str)) {
		return str;
	}

	int len = str.length();
	StringBuilder sb = new StringBuilder(str.length());
	for (int i = 0; i < len; i++) {
		char c = str.charAt(i);
		if (!Character.isWhitespace(c)) { //不为空格时追加
			sb.append(c);
		}
	}
	return sb.toString();
}

/**
 * Trim leading whitespace from the given {@code String}. 删除头部的空格：借助于StringBuilder删除指定位置的字符
 * @param str the {@code String} to check
 * @return the trimmed {@code String}
 * @see java.lang.Character#isWhitespace
 */
public static String trimLeadingWhitespace(String str) {
	if (!hasLength(str)) {
		return str;
	}

	StringBuilder sb = new StringBuilder(str);
	while (sb.length() > 0 && Character.isWhitespace(sb.charAt(0))) {
		sb.deleteCharAt(0);
	}
	return sb.toString();
}

/**
 * Trim trailing whitespace from the given {@code String}.
 * @param str the {@code String} to check
 * @return the trimmed {@code String}
 * @see java.lang.Character#isWhitespace
 */
public static String trimTrailingWhitespace(String str) {
	if (!hasLength(str)) {
		return str;
	}

	StringBuilder sb = new StringBuilder(str);
	while (sb.length() > 0 && Character.isWhitespace(sb.charAt(sb.length() - 1))) {
		sb.deleteCharAt(sb.length() - 1);
	}
	return sb.toString();
}

/**
 * Trim all occurrences of the supplied leading character from the given {@code String}.
 * @param str the {@code String} to check
 * @param leadingCharacter the leading character to be trimmed
 * @return the trimmed {@code String}
 */
public static String trimLeadingCharacter(String str, char leadingCharacter) {
	if (!hasLength(str)) {
		return str;
	}

	StringBuilder sb = new StringBuilder(str);
	while (sb.length() > 0 && sb.charAt(0) == leadingCharacter) {
		sb.deleteCharAt(0);
	}
	return sb.toString();
}

/**
 * Trim all occurrences of the supplied trailing character from the given {@code String}.
 * @param str the {@code String} to check
 * @param trailingCharacter the trailing character to be trimmed
 * @return the trimmed {@code String}
 */
public static String trimTrailingCharacter(String str, char trailingCharacter) {
	if (!hasLength(str)) {
		return str;
	}

	StringBuilder sb = new StringBuilder(str);
	while (sb.length() > 0 && sb.charAt(sb.length() - 1) == trailingCharacter) {
		sb.deleteCharAt(sb.length() - 1);
	}
	return sb.toString();
}

/**
 * Test if the given {@code String} starts with the specified prefix, 测试给定字符串是否已指定的前缀开始，忽略大小写
 * ignoring upper/lower case.
 * @param str the {@code String} to check
 * @param prefix the prefix to look for
 * @see java.lang.String#startsWith
 */
public static boolean startsWithIgnoreCase(@Nullable String str, @Nullable String prefix) {
	return (str != null && prefix != null && str.length() >= prefix.length() &&
			str.regionMatches(true, 0, prefix, 0, prefix.length())); //测试两个字符串区域是否相等
}

/**
 * Test if the given {@code String} ends with the specified suffix,测试给定字符串是否已指定的后缀结尾
 * ignoring upper/lower case.
 * @param str the {@code String} to check
 * @param suffix the suffix to look for
 * @see java.lang.String#endsWith
 */
public static boolean endsWithIgnoreCase(@Nullable String str, @Nullable String suffix) {
	return (str != null && suffix != null && str.length() >= suffix.length() &&
			str.regionMatches(true, str.length() - suffix.length(), suffix, 0, suffix.length()));
}

/**
 * Test whether the given string matches the given substring 测试给定的字符串使用匹配从指定的索引开始的子字符串
 * at the given index.
 * @param str the original string (or StringBuilder)
 * @param index the index in the original string to start matching against
 * @param substring the substring to match at the given index
 */
public static boolean substringMatch(CharSequence str, int index, CharSequence substring) {
	if (index + substring.length() > str.length()) {
		return false;
	}
	for (int i = 0; i < substring.length(); i++) {
		if (str.charAt(index + i) != substring.charAt(i)) {
			return false;
		}
	}
	return true;
}

/**
 * Count the occurrences of the substring {@code sub} in string {@code str}. 子字符串在字符串中出现的次数
 * @param str string to search in
 * @param sub string to search for
 */
public static int countOccurrencesOf(String str, String sub) {
	if (!hasLength(str) || !hasLength(sub)) {
		return 0;
	}

	int count = 0;
	int pos = 0;
	int idx;
	while ((idx = str.indexOf(sub, pos)) != -1) {
		++count;
		pos = idx + sub.length();
	}
	return count;
}

/**
 * Replace all occurrences of a substring within a string with another string. 使用其他字符串替换当前字符串中出现的全部子字符串
 * @param inString {@code String} to examine
 * @param oldPattern {@code String} to replace
 * @param newPattern {@code String} to insert
 * @return a {@code String} with the replacements
 */
public static String replace(String inString, String oldPattern, @Nullable String newPattern) {
	if (!hasLength(inString) || !hasLength(oldPattern) || newPattern == null) {
		return inString;
	}
	int index = inString.indexOf(oldPattern);
	if (index == -1) {
		// no occurrence -> can return input as-is
		return inString;
	}

	int capacity = inString.length();
	if (newPattern.length() > oldPattern.length()) {
		capacity += 16;
	}
	StringBuilder sb = new StringBuilder(capacity);

	int pos = 0;  // our position in the old string 旧字符串位置
	int patLen = oldPattern.length();
	while (index >= 0) {
		sb.append(inString.substring(pos, index)); //截取
		sb.append(newPattern); //缀加
		pos = index + patLen; //后移位置
		index = inString.indexOf(oldPattern, pos); //再次获取出现的位置
	}

	// append any characters to the right of a match 缀加右侧未匹配的所有字符串
	sb.append(inString.substring(pos));
	return sb.toString();
}

/**
 * Delete all occurrences of the given substring. 删除给定的子字符串
 * @param inString the original {@code String}
 * @param pattern the pattern to delete all occurrences of
 * @return the resulting {@code String}
 */
public static String delete(String inString, String pattern) {
	return replace(inString, pattern, "");
}

/**
 * Delete any character in a given {@code String}. 删除给定字符串的所有匹配的字符
 * @param inString the original {@code String}
 * @param charsToDelete a set of characters to delete.
 * E.g. "az\n" will delete 'a's, 'z's and new lines.
 * @return the resulting {@code String}
 */
public static String deleteAny(String inString, @Nullable String charsToDelete) {
	if (!hasLength(inString) || !hasLength(charsToDelete)) {
		return inString;
	}

	StringBuilder sb = new StringBuilder(inString.length());
	for (int i = 0; i < inString.length(); i++) {
		char c = inString.charAt(i);
		if (charsToDelete.indexOf(c) == -1) { //不包括时
			sb.append(c);
		}
	}
	return sb.toString();
}


```

### 格式化字符串

```java
/**
 * Quote the given {@code String} with single quotes. 单引号引用字符串
 * @param str the input {@code String} (e.g. "myString")
 * @return the quoted {@code String} (e.g. "'myString'"),
 * or {@code null} if the input was {@code null}
 */
@Nullable
public static String quote(@Nullable String str) {
	return (str != null ? "'" + str + "'" : null);
}

/**
 * Turn the given Object into a {@code String} with single quotes
 * if it is a {@code String}; keeping the Object as-is else.
 * @param obj the input Object (e.g. "myString")
 * @return the quoted {@code String} (e.g. "'myString'"),
 * or the input object as-is if not a {@code String}
 */
@Nullable
public static Object quoteIfString(@Nullable Object obj) {
	return (obj instanceof String ? quote((String) obj) : obj);
}

/**
 * Unqualify a string qualified by a '.' dot character. For example, 取消限定一个符合"."条件的字符串
 * "this.name.is.qualified", returns "qualified".
 * @param qualifiedName the qualified name
 */
public static String unqualify(String qualifiedName) {
	return unqualify(qualifiedName, '.');
}

/**
 * Unqualify a string qualified by a separator character(指定字符串). For example, (从后面截取出现的字符)
 * "this:name:is:qualified" returns "qualified" if using a ':' separator.
 * @param qualifiedName the qualified name
 * @param separator the separator
 */
public static String unqualify(String qualifiedName, char separator) {
	return qualifiedName.substring(qualifiedName.lastIndexOf(separator) + 1);
}

/**
 * Capitalize a {@code String}, changing the first letter to 第一个字母大写
 * upper case as per {@link Character#toUpperCase(char)}.
 * No other letters are changed. 其他字母不变
 * @param str the {@code String} to capitalize
 * @return the capitalized {@code String}
 */
public static String capitalize(String str) {
	return changeFirstCharacterCase(str, true);
}

/**
 * Uncapitalize a {@code String}, changing the first letter to 第一个字母小写
 * lower case as per {@link Character#toLowerCase(char)}.
 * No other letters are changed.
 * @param str the {@code String} to uncapitalize
 * @return the uncapitalized {@code String}
 */
public static String uncapitalize(String str) {
	return changeFirstCharacterCase(str, false);
}

private static String changeFirstCharacterCase(String str, boolean capitalize) {
	if (!hasLength(str)) {
		return str;
	}

	char baseChar = str.charAt(0);
	char updatedChar;
	if (capitalize) {
		updatedChar = Character.toUpperCase(baseChar);
	}
	else {
		updatedChar = Character.toLowerCase(baseChar);
	}
	if (baseChar == updatedChar) { //自身本来就是大写或者小写
		return str;
	}

	char[] chars = str.toCharArray();
	chars[0] = updatedChar;
	return new String(chars, 0, chars.length);
}

/**
 * Extract the filename from the given Java resource path, 提取给定资源路径的文件名，不包括文件目录
 * e.g. {@code "mypath/myfile.txt" -> "myfile.txt"}.
 * @param path the file path (may be {@code null})
 * @return the extracted filename, or {@code null} if none
 */
@Nullable
public static String getFilename(@Nullable String path) {
	if (path == null) {
		return null;
	}

	int separatorIndex = path.lastIndexOf(FOLDER_SEPARATOR); 
	return (separatorIndex != -1 ? path.substring(separatorIndex + 1) : path);
}

/**
 * Extract the filename extension from the given Java resource path, 提取给定资源路径的文件名后缀
 * e.g. "mypath/myfile.txt" -> "txt".
 * @param path the file path (may be {@code null})
 * @return the extracted filename extension, or {@code null} if none
 */
@Nullable
public static String getFilenameExtension(@Nullable String path) {
	if (path == null) {
		return null;
	}

	int extIndex = path.lastIndexOf(EXTENSION_SEPARATOR);
	if (extIndex == -1) {
		return null;
	}

	int folderIndex = path.lastIndexOf(FOLDER_SEPARATOR);
	if (folderIndex > extIndex) {
		return null;
	}

	return path.substring(extIndex + 1);
}

/**
 * Strip the filename extension from the given Java resource path, 删除文件名后缀
 * e.g. "mypath/myfile.txt" -> "mypath/myfile".
 * @param path the file path
 * @return the path with stripped filename extension
 */
public static String stripFilenameExtension(String path) {
	int extIndex = path.lastIndexOf(EXTENSION_SEPARATOR);
	if (extIndex == -1) {
		return path;
	}

	int folderIndex = path.lastIndexOf(FOLDER_SEPARATOR);
	if (folderIndex > extIndex) {
		return path;
	}

	return path.substring(0, extIndex);
}

/**
 * Apply the given relative path to the given Java resource path,将给定的相对路径应用于给定的Java资源路径
 * assuming standard Java folder separation (i.e. "/" separators).
 * @param path the path to start from (usually a full file path)
 * @param relativePath the relative path to apply
 * (relative to the full file path above)
 * @return the full file path that results from applying the relative path 应用相对路径生成的完整文件路径
 */
public static String applyRelativePath(String path, String relativePath) {
	int separatorIndex = path.lastIndexOf(FOLDER_SEPARATOR);
	if (separatorIndex != -1) {
		String newPath = path.substring(0, separatorIndex);
		if (!relativePath.startsWith(FOLDER_SEPARATOR)) {
			newPath += FOLDER_SEPARATOR;
		}
		return newPath + relativePath;
	}
	else {
		return relativePath;
	}
}

/**
 * Normalize the path by suppressing sequences like "path/.." and 标准化文件路径
 * inner simple dots.
 * <p>The result is convenient for path comparison(结果便于路径比较). For other uses(其他用途),
 * notice that Windows separators ("\") are replaced by simple slashes.Windows的分隔符使用“/”代替
 * @param path the original path
 * @return the normalized path
 */
public static String cleanPath(String path) {
	if (!hasLength(path)) {
		return path;
	}
	String pathToUse = replace(path, WINDOWS_FOLDER_SEPARATOR, FOLDER_SEPARATOR); //替换windows分隔符

	// Strip prefix from path to analyze(提取前缀进行分析), to not treat it as part of the
	// first path element(不要把前缀当做第一个路径元素). This is necessary to correctly parse paths(这对于正确解析路径是必要的) like
	// "file:core/../core/io/Resource.class", where the ".." should just
	// strip the first "core" directory while keeping the "file:" prefix.
	int prefixIndex = pathToUse.indexOf(':'); //获取":"位置
	String prefix = "";
	if (prefixIndex != -1) {
		prefix = pathToUse.substring(0, prefixIndex + 1); //获取前缀，如"file:"
		if (prefix.contains(FOLDER_SEPARATOR)) { // 包含"/",前缀为空
			prefix = ""; 
		}
		else {
			pathToUse = pathToUse.substring(prefixIndex + 1); ":" //后面的使用路径
		}
	}
	if (pathToUse.startsWith(FOLDER_SEPARATOR)) { //使用路径以"/"开始时，前缀添加"/" ，使用路径取消"/"
		prefix = prefix + FOLDER_SEPARATOR;
		pathToUse = pathToUse.substring(1);
	}

	String[] pathArray = delimitedListToStringArray(pathToUse, FOLDER_SEPARATOR); //以"/"分割路径
	LinkedList<String> pathElements = new LinkedList<>();  //用于储存clean 后的path 元素，不包括".."、"."
	int tops = 0; //用于记录有“..”的个数

	//从后往前处理：找对顶部路径元素
	for (int i = pathArray.length - 1; i >= 0; i--) {
		String element = pathArray[i];
		if (CURRENT_PATH.equals(element)) { //当前路径"."
			// Points to current directory - drop it. 指向当前目录-删除它
		}
		else if (TOP_PATH.equals(element)) { //顶部路径":"
			// Registering top path found. //注册找到的顶路径
			tops++; //记录下“..”已出现的个数
		}
		else {
			if (tops > 0) {
				// Merging path element with element corresponding to top path. 将路径元素与对应于顶部路径的元素合并
				tops--; //..个数减一
			}
			else {
				// Normal path element found. //合法元素添加到链表中
				pathElements.add(0, element);
			}
		}
	}

	// Remaining top paths need to be retained. 需要保留其余的顶级路径
	for (int i = 0; i < tops; i++) {
		pathElements.add(0, TOP_PATH);
	}
	// If nothing else left, at least explicitly point to current path. //如果没有其他内容，至少显式地指向当前路径
	if (pathElements.size() == 1 && "".equals(pathElements.getLast()) && !prefix.endsWith(FOLDER_SEPARATOR)) {
		pathElements.add(0, CURRENT_PATH);
	}

	return prefix + collectionToDelimitedString(pathElements, FOLDER_SEPARATOR);
}

/**
 * Compare two paths after normalization of them. 标准化路径后是否相等
 * @param path1 first path for comparison
 * @param path2 second path for comparison
 * @return whether the two paths are equivalent after normalization
 */
public static boolean pathEquals(String path1, String path2) {
	return cleanPath(path1).equals(cleanPath(path2));
}

/**
 * Decode the given encoded URI component value(解码给定编码的URI组件的值). Based on the following rules:
 * <ul>
 * <li>Alphanumeric characters {@code "a"} through {@code "z"}, {@code "A"} through {@code "Z"},
 * and {@code "0"} through {@code "9"} stay the same.</li> [a-z A-Z 0-9] [- _ . *]不变
 * <li>Special characters {@code "-"}, {@code "_"}, {@code "."}, and {@code "*"} stay the same.</li>
 * <li>A sequence "{@code %<i>xy</i>}" is interpreted as a hexadecimal representation of the character. (解释为字符的十六进制表示)</li>
 * </ul>
 * @param source the encoded String
 * @param charset the character set
 * @return the decoded value
 * @throws IllegalArgumentException when the given source contains invalid encoded sequences
 * @since 5.0
 * @see java.net.URLDecoder#decode(String, String)
 */
public static String uriDecode(String source, Charset charset) {
	int length = source.length();
	if (length == 0) {
		return source;
	}
	Assert.notNull(charset, "Charset must not be null");

	ByteArrayOutputStream bos = new ByteArrayOutputStream(length); //字节输出输出流
	boolean changed = false;
	for (int i = 0; i < length; i++) { //遍历每个字符串
		int ch = source.charAt(i);
		if (ch == '%') {  //uri encode 编码中使用%开始表示特殊字符，将其转化为16进制：一个字符占两个字节
			if (i + 2 < length) {
				char hex1 = source.charAt(i + 1); 
				char hex2 = source.charAt(i + 2);
				int u = Character.digit(hex1, 16); //高位
				int l = Character.digit(hex2, 16); //地位
				if (u == -1 || l == -1) {
					throw new IllegalArgumentException("Invalid encoded sequence \"" + source.substring(i) + "\"");
				}
				bos.write((char) ((u << 4) + l)); //16进制字字节：高位左移四位+地位
				i += 2;
				changed = true;
			}
			else {
				throw new IllegalArgumentException("Invalid encoded sequence \"" + source.substring(i) + "\"");
			}
		}
		else {
			bos.write(ch);
		}
	}
	return (changed ? new String(bos.toByteArray(), charset) : source); //使用指定的字符串编码
}

/**
 * Parse the given {@code String} value into a {@link Locale}, accepting 解析给定的字符串为Local
 * the {@link Locale#toString} format as well as BCP 47 language tags.
 * @param localeValue the locale value: following either {@code Locale's}
 * {@code toString()} format ("en", "en_UK", etc), also accepting spaces as
 * separators (as an alternative to underscores), or BCP 47 (e.g. "en-UK")
 * as specified by {@link Locale#forLanguageTag} on Java 7+
 * @return a corresponding {@code Locale} instance, or {@code null} if none
 * @throws IllegalArgumentException in case of an invalid locale specification
 * @since 5.0.4
 * @see #parseLocaleString
 * @see Locale#forLanguageTag
 */
@Nullable
public static Locale parseLocale(String localeValue) {
	String[] tokens = tokenizeLocaleSource(localeValue);
	if (tokens.length == 1) {
		Locale resolved = Locale.forLanguageTag(localeValue);
		return (resolved.getLanguage().length() > 0 ? resolved : null);
	}
	return parseLocaleTokens(localeValue, tokens);
}

/**
 * Parse the given {@code String} representation into a {@link Locale}.
 * <p>For many parsing scenarios(对于许多解析场景), this is an inverse operation of
 * {@link Locale#toString Locale's toString}, in a lenient sense.
 * This method does not aim for strict {@code Locale} design compliance;
 * it is rather specifically tailored for typical Spring parsing needs.
 * <p><b>Note: This delegate does not accept the BCP 47 language tag format.
 * Please use {@link #parseLocale} for lenient parsing of both formats.</b>
 * @param localeString the locale {@code String}: following {@code Locale's}
 * {@code toString()} format ("en", "en_UK", etc), also accepting spaces as
 * separators (as an alternative to underscores)
 * @return a corresponding {@code Locale} instance, or {@code null} if none
 * @throws IllegalArgumentException in case of an invalid locale specification
 */
@Nullable
public static Locale parseLocaleString(String localeString) {
	return parseLocaleTokens(localeString, tokenizeLocaleSource(localeString));
}

private static String[] tokenizeLocaleSource(String localeSource) {
	return tokenizeToStringArray(localeSource, "_ ", false, false);
}

@Nullable
private static Locale parseLocaleTokens(String localeString, String[] tokens) {
	String language = (tokens.length > 0 ? tokens[0] : "");
	String country = (tokens.length > 1 ? tokens[1] : "");
	validateLocalePart(language);
	validateLocalePart(country);

	String variant = "";
	if (tokens.length > 2) {
		// There is definitely a variant, and it is everything after the country
		// code sans the separator between the country code and the variant.
		int endIndexOfCountryCode = localeString.indexOf(country, language.length()) + country.length();
		// Strip off any leading '_' and whitespace, what's left is the variant.
		variant = trimLeadingWhitespace(localeString.substring(endIndexOfCountryCode));
		if (variant.startsWith("_")) {
			variant = trimLeadingCharacter(variant, '_');
		}
	}

	if ("".equals(variant) && country.startsWith("#")) {
		variant = country;
		country = "";
	}

	return (language.length() > 0 ? new Locale(language, country, variant) : null);
}

private static void validateLocalePart(String localePart) {
	for (int i = 0; i < localePart.length(); i++) {
		char ch = localePart.charAt(i);
		if (ch != ' ' && ch != '_' && ch != '#' && !Character.isLetterOrDigit(ch)) {
			throw new IllegalArgumentException(
					"Locale part \"" + localePart + "\" contains invalid characters");
		}
	}
}

/**
 * Determine the RFC 3066 compliant language tag(确定符合RFC 3066的语言标记),
 * as used for the HTTP "Accept-Language" header.
 * @param locale the Locale to transform to a language tag
 * @return the RFC 3066 compliant language tag as {@code String}
 * @deprecated as of 5.0.4, in favor of {@link Locale#toLanguageTag()}
 */
@Deprecated
public static String toLanguageTag(Locale locale) {
	return locale.getLanguage() + (hasText(locale.getCountry()) ? "-" + locale.getCountry() : "");
}

/**
 * Parse the given {@code timeZoneString} value into a {@link TimeZone}. 解析给定的timeZoneString为TimeZone时区
 * @param timeZoneString the time zone {@code String}, following {@link TimeZone#getTimeZone(String)}
 * but throwing {@link IllegalArgumentException} in case of an invalid time zone specification
 * @return a corresponding {@link TimeZone} instance
 * @throws IllegalArgumentException in case of an invalid time zone specification
 */
public static TimeZone parseTimeZoneString(String timeZoneString) {
	TimeZone timeZone = TimeZone.getTimeZone(timeZoneString);
	if ("GMT".equals(timeZone.getID()) && !timeZoneString.startsWith("GMT")) {
		// We don't want that GMT fallback...
		throw new IllegalArgumentException("Invalid time zone specification '" + timeZoneString + "'");
	}
	return timeZone;
}

```

### 处理字符串数组

```java
/**
 * Append the given {@code String} to the given {@code String} array,将给定的str缀加到数组中array
 * returning a new array consisting of the input array contents plus 返回新的数组
 * the given {@code String}.
 * @param array the array to append to (can be {@code null})
 * @param str the {@code String} to append
 * @return the new array (never {@code null})
 */
public static String[] addStringToArray(@Nullable String[] array, String str) {
	if (ObjectUtils.isEmpty(array)) {
		return new String[] {str};
	}

	String[] newArr = new String[array.length + 1]; //复制、copy到新的数组中
	System.arraycopy(array, 0, newArr, 0, array.length);
	newArr[array.length] = str;
	return newArr;
}

/**
 * Concatenate the given {@code String} arrays into one, 连接两个数组元素
 * with overlapping array elements included twice.
 * <p>The order of elements in the original arrays is preserved.
 * @param array1 the first array (can be {@code null})
 * @param array2 the second array (can be {@code null})
 * @return the new array ({@code null} if both given arrays were {@code null})
 */
@Nullable
public static String[] concatenateStringArrays(@Nullable String[] array1, @Nullable String[] array2) {
	if (ObjectUtils.isEmpty(array1)) {
		return array2;
	}
	if (ObjectUtils.isEmpty(array2)) {
		return array1;
	}

	String[] newArr = new String[array1.length + array2.length];
	System.arraycopy(array1, 0, newArr, 0, array1.length);
	System.arraycopy(array2, 0, newArr, array1.length, array2.length);
	return newArr;
}

/**
 * Merge the given {@code String} arrays into one, with overlapping 合并给定的数组到一个数组中，
 * array elements only included once(数组元素只包含一次).
 * <p>The order of elements in the original arrays(原始数组的元素顺序) is preserved 保持不变
 * (with the exception of overlapping elements除了重叠的元素, which are only
 * included on their first occurrence).
 * @param array1 the first array (can be {@code null})
 * @param array2 the second array (can be {@code null})
 * @return the new array ({@code null} if both given arrays were {@code null})
 * @deprecated as of 4.3.15, in favor of manual merging via {@link LinkedHashSet}
 * (with every entry included at most once, even entries within the first array)
 */
@Deprecated
@Nullable
public static String[] mergeStringArrays(@Nullable String[] array1, @Nullable String[] array2) {
	if (ObjectUtils.isEmpty(array1)) {
		return array2;
	}
	if (ObjectUtils.isEmpty(array2)) {
		return array1;
	}

	List<String> result = new ArrayList<>();
	result.addAll(Arrays.asList(array1));
	for (String str : array2) {
		if (!result.contains(str)) {
			result.add(str);
		}
	}
	return toStringArray(result);
}

/**
 * Turn given source {@code String} array into sorted array. 排序
 * @param array the source array
 * @return the sorted array (never {@code null})
 */
public static String[] sortStringArray(String[] array) {
	if (ObjectUtils.isEmpty(array)) {
		return new String[0];
	}

	Arrays.sort(array);
	return array;
}

/**
 * Copy the given {@code Collection} into a {@code String} array. 将集合转换为数组
 * <p>The {@code Collection} must contain {@code String} elements only.
 * @param collection the {@code Collection} to copy
 * @return the {@code String} array
 */
public static String[] toStringArray(Collection<String> collection) {
	return collection.toArray(new String[0]);
}

/**
 * Copy the given Enumeration into a {@code String} array. 将枚举列表转换为数组
 * The Enumeration must contain {@code String} elements only.
 * @param enumeration the Enumeration to copy
 * @return the {@code String} array
 */
public static String[] toStringArray(Enumeration<String> enumeration) {
	return toStringArray(Collections.list(enumeration));
}

/**
 * Trim the elements of the given {@code String} array,trim指定数组的元素
 * calling {@code String.trim()} on each of them.
 * @param array the original {@code String} array (potentially empty)
 * @return the resulting array (of the same size) with trimmed elements
 */
public static String[] trimArrayElements(@Nullable String[] array) {
	if (ObjectUtils.isEmpty(array)) {
		return new String[0];
	}

	String[] result = new String[array.length];
	for (int i = 0; i < array.length; i++) {
		String element = array[i];
		result[i] = (element != null ? element.trim() : null);
	}
	return result;
}

/**
 * Remove duplicate strings from the given array. 删除重复
 * <p>As of 4.2, it preserves the original order, as it uses a {@link LinkedHashSet}.
 * @param array the {@code String} array (potentially empty)
 * @return an array without duplicates, in natural sort order 自然顺序排列
 */
public static String[] removeDuplicateStrings(String[] array) {
	if (ObjectUtils.isEmpty(array)) {
		return array;
	}

	Set<String> set = new LinkedHashSet<>(Arrays.asList(array));
	return toStringArray(set);
}

/**
 * Split a {@code String} at the first occurrence of the delimiter. 分割字符在第一次出现时
 * Does not include the delimiter in the result. 结果中不包含分隔符
 * @param toSplit the string to split (potentially {@code null} or empty)
 * @param delimiter to split the string up with (potentially {@code null} or empty)
 * @return a two element array(两个元素) with index 0 being before the delimiter, and
 * index 1 being after the delimiter (neither element includes the delimiter);
 * or {@code null} if the delimiter wasn't found in the given input {@code String} 如果没有delimiter，返回null
 */
@Nullable
public static String[] split(@Nullable String toSplit, @Nullable String delimiter) {
	if (!hasLength(toSplit) || !hasLength(delimiter)) {
		return null;
	}
	int offset = toSplit.indexOf(delimiter);
	if (offset < 0) {
		return null;
	}

	String beforeDelimiter = toSplit.substring(0, offset);
	String afterDelimiter = toSplit.substring(offset + delimiter.length());
	return new String[] {beforeDelimiter, afterDelimiter};
}

/**
 * Take an array of strings and split each element based on the given delimiter.
 * A {@code Properties} instance is then generated, with the left of the delimiter
 * providing the key, and the right of the delimiter providing the value.
 * <p>Will trim both the key and value before adding them to the {@code Properties}.
 * @param array the array to process
 * @param delimiter to split each element using (typically the equals symbol)
 * @return a {@code Properties} instance representing the array contents,
 * or {@code null} if the array to process was {@code null} or empty
 */
@Nullable
public static Properties splitArrayElementsIntoProperties(String[] array, String delimiter) {
	return splitArrayElementsIntoProperties(array, delimiter, null);
}

/**
 * Take an array of strings and split each element based on the given delimiter(取一个字符串数组，根据给定的分隔符拆分每个元素).
 * A {@code Properties} instance is then generated(生成一个Properties实例), with the left of the
 * delimiter providing the key, and the right of the delimiter providing the value. 分隔符左边为key，右边为值
 * <p>Will trim both the key and value before adding them to the 添加前trem key value
 * {@code Properties} instance.
 * @param array the array to process
 * @param delimiter to split each element using (typically the equals symbol)
 * @param charsToDelete one or more characters to remove from each element
 * prior to attempting the split operation (typically the quotation mark
 * symbol), or {@code null} if no removal should occur
 * @return a {@code Properties} instance representing the array contents,
 * or {@code null} if the array to process was {@code null} or empty
 */
@Nullable
public static Properties splitArrayElementsIntoProperties(
		String[] array, String delimiter, @Nullable String charsToDelete) {

	if (ObjectUtils.isEmpty(array)) {
		return null;
	}

	Properties result = new Properties();
	for (String element : array) {
		if (charsToDelete != null) {
			element = deleteAny(element, charsToDelete);
		}
		String[] splittedElement = split(element, delimiter);
		if (splittedElement == null) {
			continue;
		}
		result.setProperty(splittedElement[0].trim(), splittedElement[1].trim());
	}
	return result;
}

/**
 * Tokenize the given {@code String} into a {@code String} array via a
 * {@link StringTokenizer}.
 * <p>Trims tokens and omits empty tokens.
 * <p>The given {@code delimiters} string can consist of any number of
 * delimiter characters. Each of those characters can be used to separate
 * tokens. A delimiter is always a single character; for multi-character
 * delimiters, consider using {@link #delimitedListToStringArray}.
 * @param str the {@code String} to tokenize (potentially {@code null} or empty)
 * @param delimiters the delimiter characters, assembled as a {@code String}
 * (each of the characters is individually considered as a delimiter)
 * @return an array of the tokens
 * @see java.util.StringTokenizer
 * @see String#trim()
 * @see #delimitedListToStringArray
 */
public static String[] tokenizeToStringArray(@Nullable String str, String delimiters) {
	return tokenizeToStringArray(str, delimiters, true, true);
}

/**
 * Tokenize the given {@code String} into a {@code String} array via a
 * {@link StringTokenizer}.
 * <p>The given {@code delimiters} string can consist of any number of
 * delimiter characters. Each of those characters can be used to separate
 * tokens. A delimiter is always a single character; for multi-character
 * delimiters, consider using {@link #delimitedListToStringArray}.
 * @param str the {@code String} to tokenize (potentially {@code null} or empty)
 * @param delimiters the delimiter characters, assembled as a {@code String}
 * (each of the characters is individually considered as a delimiter)
 * @param trimTokens trim the tokens via {@link String#trim()} trim标记
 * @param ignoreEmptyTokens omit empty tokens from the result array 从结果数组中省略空标记
 * (only applies to tokens that are empty after trimming; StringTokenizer
 * will not consider subsequent delimiters as token in the first place).
 * @return an array of the tokens
 * @see java.util.StringTokenizer
 * @see String#trim()
 * @see #delimitedListToStringArray
 */
public static String[] tokenizeToStringArray(
		@Nullable String str, String delimiters, boolean trimTokens, boolean ignoreEmptyTokens) {

	if (str == null) {
		return new String[0];
	}

	StringTokenizer st = new StringTokenizer(str, delimiters); //使用jdk 提供的StringTokenizer 字符串分词器
	List<String> tokens = new ArrayList<>();
	while (st.hasMoreTokens()) {
		String token = st.nextToken();
		if (trimTokens) {
			token = token.trim();
		}
		if (!ignoreEmptyTokens || token.length() > 0) {
			tokens.add(token);
		}
	}
	return toStringArray(tokens);
}

/**
 * Take a {@code String} that is a delimited list and convert it into a
 * {@code String} array.
 * <p>A single {@code delimiter} may consist of more than one character,
 * but it will still be considered as a single delimiter string, rather
 * than as bunch of potential delimiter characters, in contrast to
 * {@link #tokenizeToStringArray}.
 * @param str the input {@code String} (potentially {@code null} or empty)
 * @param delimiter the delimiter between elements (this is a single delimiter,
 * rather than a bunch individual delimiter characters)
 * @return an array of the tokens in the list
 * @see #tokenizeToStringArray
 */
public static String[] delimitedListToStringArray(@Nullable String str, @Nullable String delimiter) {
	return delimitedListToStringArray(str, delimiter, null);
}

/**
 * Take a {@code String} that is a delimited list and convert it into
 * a {@code String} array.
 * <p>A single {@code delimiter} may consist of more than one character,
 * but it will still be considered as a single delimiter string, rather
 * than as bunch of potential delimiter characters, in contrast to
 * {@link #tokenizeToStringArray}.
 * @param str the input {@code String} (potentially {@code null} or empty)
 * @param delimiter the delimiter between elements (this is a single delimiter,
 * rather than a bunch individual delimiter characters)
 * @param charsToDelete a set of characters to delete(要删除的一组字符); useful for deleting unwanted
 * line breaks: e.g. "\r\n\f" will delete all new lines and line feeds in a {@code String}
 * @return an array of the tokens in the list
 * @see #tokenizeToStringArray
 */
public static String[] delimitedListToStringArray(
		@Nullable String str, @Nullable String delimiter, @Nullable String charsToDelete) {

	if (str == null) {
		return new String[0];
	}
	if (delimiter == null) {
		return new String[] {str};
	}

	List<String> result = new ArrayList<>();
	if ("".equals(delimiter)) {
		for (int i = 0; i < str.length(); i++) {
			result.add(deleteAny(str.substring(i, i + 1), charsToDelete));
		}
	}
	else {
		int pos = 0;
		int delPos;
		while ((delPos = str.indexOf(delimiter, pos)) != -1) {
			result.add(deleteAny(str.substring(pos, delPos), charsToDelete));
			pos = delPos + delimiter.length();
		}
		if (str.length() > 0 && pos <= str.length()) {
			// Add rest of String, but not in case of empty input.
			result.add(deleteAny(str.substring(pos), charsToDelete));
		}
	}
	return toStringArray(result);
}

/**
 * Convert a comma delimited list (e.g., a row from a CSV file) into an
 * array of strings.
 * @param str the input {@code String} (potentially {@code null} or empty)
 * @return an array of strings, or the empty array in case of empty input
 */
public static String[] commaDelimitedListToStringArray(@Nullable String str) {
	return delimitedListToStringArray(str, ",");
}

/**
 * Convert a comma delimited list (e.g., a row from a CSV file) into a set.
 * <p>Note that this will suppress duplicates, and as of 4.2, the elements in
 * the returned set will preserve the original order in a {@link LinkedHashSet}.
 * @param str the input {@code String} (potentially {@code null} or empty)
 * @return a set of {@code String} entries in the list
 * @see #removeDuplicateStrings(String[])
 */
public static Set<String> commaDelimitedListToSet(@Nullable String str) {
	String[] tokens = commaDelimitedListToStringArray(str);
	return new LinkedHashSet<>(Arrays.asList(tokens));
}

/**
 * Convert a {@link Collection} to a delimited {@code String} (e.g. CSV).
 * <p>Useful for {@code toString()} implementations.
 * @param coll the {@code Collection} to convert (potentially {@code null} or empty) 
 * @param delim the delimiter to use (typically a ",") 分隔符
 * @param prefix the {@code String} to start each element with 前缀
 * @param suffix the {@code String} to end each element with 后缀
 * @return the delimited {@code String}
 */
public static String collectionToDelimitedString(
		@Nullable Collection<?> coll, String delim, String prefix, String suffix) {

	if (CollectionUtils.isEmpty(coll)) {
		return "";
	}

	StringBuilder sb = new StringBuilder();
	Iterator<?> it = coll.iterator();
	while (it.hasNext()) {
		sb.append(prefix).append(it.next()).append(suffix);
		if (it.hasNext()) {
			sb.append(delim);
		}
	}
	return sb.toString();
}

/**
 * Convert a {@code Collection} into a delimited {@code String} (e.g. CSV).
 * <p>Useful for {@code toString()} implementations.
 * @param coll the {@code Collection} to convert (potentially {@code null} or empty)
 * @param delim the delimiter to use (typically a ",")
 * @return the delimited {@code String}
 */
public static String collectionToDelimitedString(@Nullable Collection<?> coll, String delim) {
	return collectionToDelimitedString(coll, delim, "", "");
}

/**
 * Convert a {@code Collection} into a delimited {@code String} (e.g., CSV). 将集合使用“,”分割转换为String
 * <p>Useful for {@code toString()} implementations.
 * @param coll the {@code Collection} to convert (potentially {@code null} or empty)
 * @return the delimited {@code String}
 */
public static String collectionToCommaDelimitedString(@Nullable Collection<?> coll) {
	return collectionToDelimitedString(coll, ",");
}

/**
 * Convert a {@code String} array into a delimited {@code String} (e.g. CSV). 将arr转换为使用delim分隔符的字符串
 * <p>Useful for {@code toString()} implementations.
 * @param arr the array to display (potentially {@code null} or empty)
 * @param delim the delimiter to use (typically a ",")
 * @return the delimited {@code String}
 */
public static String arrayToDelimitedString(@Nullable Object[] arr, String delim) {
	if (ObjectUtils.isEmpty(arr)) {
		return "";
	}
	if (arr.length == 1) {
		return ObjectUtils.nullSafeToString(arr[0]);
	}

	StringBuilder sb = new StringBuilder();
	for (int i = 0; i < arr.length; i++) {
		if (i > 0) {
			sb.append(delim);
		}
		sb.append(arr[i]);
	}
	return sb.toString();
}

/**
 * Convert a {@code String} array into a comma delimited {@code String} ","分割数组转换为string
 * (i.e., CSV).
 * <p>Useful for {@code toString()} implementations.
 * @param arr the array to display (potentially {@code null} or empty)
 * @return the delimited {@code String}
 */
public static String arrayToCommaDelimitedString(@Nullable Object[] arr) {
	return arrayToDelimitedString(arr, ",");
}

```
