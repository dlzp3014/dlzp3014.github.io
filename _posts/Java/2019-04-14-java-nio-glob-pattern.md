---
layout: post
title:  "Java Nio中文件名操作的Glob模式"
date:   2019-04-14 11:49:00
categories: Java 
tags: Java Java-Nio
---

* content
{:toc}


Glob是一种匹配模式，使用通配符来指定文件名。例如：.java就是一个简单的Glob，它指定了所有扩展名为"java"的文件。Glob模式中广泛使用了两个通配符"*"和"?":"*"表示任意的字符或字符组成字符串，"?":表示任意单个字符。Glob模式与正则表达式类似，但它的功能有限。主要用于：

- FileSystem#getPathMatcher:public abstract PathMatcher getPathMatcher(String syntaxAndPattern) 获取Path匹配器 
- Files#newDirectoryStream(Path,String)：static DirectoryStream<Path> newDirectoryStream(Path dir, String glob)根据指定的Glob模式获取DirectoryStream<Path>，遍历目录下的文件


## 匹配规范

### 语法

- "*" 匹配0或者多个字符，不跨越目录边界

- "**" 匹配0或者多个字符，跨越目录边界

- ？ 匹配一个字符

- \ 转移字符，如"\{" 匹配 "\{"

- [] 方括号表达式，匹配一个设置的字符，如[abc]匹配 a或者b或者c; [a-z] 匹配a到z字符; [!a-c] 取反; 

- {} 组模式,使用","分割，如*.{html,css,js}



### JDK1.7 中FileSystem#getPathMatcher(String syntaxAndPattern)方法说明

```java
/**
 * Returns a {@code PathMatcher} that performs match operations on the 
 * {@code String} representation of {@link Path} objects by interpreting a
 * given pattern.
 *
 * The {@code syntaxAndPattern} parameter identifies the syntax and the
 * pattern and takes the form:
 * <blockquote><pre>
 * <i>syntax</i><b>:</b><i>pattern</i>
 * </pre></blockquote>
 * where {@code ':'} stands for itself.
 *
 * <p> A {@code FileSystem} implementation supports the "{@code glob}" and 文件系统实现支持glob和regex语法
 * "{@code regex}" syntaxes, and may support others. The value of the syntax
 * component is compared without regard to case.
 *
 * <p> When the syntax is "{@code glob}" then the {@code String}
 * representation of the path is matched using a limited pattern language
 * that resembles regular expressions but with a simpler syntax. For example:
 *
 * <blockquote>
 * <table border="0" summary="Pattern Language">
 * <tr>
 *   <td>{@code *.java}</td>
 *   <td>Matches a path that represents a file name ending in {@code .java}</td>
 * </tr>
 * <tr>
 *   <td>{@code *.*}</td>
 *   <td>Matches file names containing a dot</td>
 * </tr>
 * <tr>
 *   <td>{@code *.{java,class}}</td>
 *   <td>Matches file names ending with {@code .java} or {@code .class}</td>
 * </tr>
 * <tr>
 *   <td>{@code foo.?}</td>
 *   <td>Matches file names starting with {@code foo.} and a single
 *   character extension</td>
 * </tr>
 * <tr>
 *   <td><tt>&#47;home&#47;*&#47;*</tt>
 *   <td>Matches <tt>&#47;home&#47;gus&#47;data</tt> on UNIX platforms</td>
 * </tr>
 * <tr>
 *   <td><tt>&#47;home&#47;**</tt>
 *   <td>Matches <tt>&#47;home&#47;gus</tt> and
 *   <tt>&#47;home&#47;gus&#47;data</tt> on UNIX platforms</td>
 * </tr>
 * <tr>
 *   <td><tt>C:&#92;&#92;*</tt>
 *   <td>Matches <tt>C:&#92;foo</tt> and <tt>C:&#92;bar</tt> on the Windows
 *   platform (note that the backslash is escaped; as a string literal in the
 *   Java Language the pattern would be <tt>"C:&#92;&#92;&#92;&#92;*"</tt>) </td>
 * </tr>
 *
 * </table>
 * </blockquote>
 *
 * <p> The following rules are used to interpret glob patterns:
 *
 * <ul>
 *   <li><p> The {@code *} character matches zero or more {@link Character
 *   characters} of a {@link Path#getName(int) name} component without
 *   crossing directory boundaries. </p></li>
 *
 *   <li><p> The {@code **} characters matches zero or more {@link Character
 *   characters} crossing directory boundaries. </p></li>
 *
 *   <li><p> The {@code ?} character matches exactly one character of a
 *   name component.</p></li>
 *
 *   <li><p> The backslash character ({@code \}) is used to escape characters 转义字符
 *   that would otherwise be interpreted as special characters. The expression
 *   {@code \\} matches a single backslash and "\{" matches a left brace
 *   for example.  </p></li>
 *
 *   <li><p> The {@code [ ]} characters are a <i>bracket expression</i> that []表达式
 *   match a single character of a name component out of a set of characters.
 *   For example, {@code [abc]} matches {@code "a"}, {@code "b"}, or {@code "c"}.
 *   The hyphen ({@code -}) may be used to specify a range so {@code [a-z]}
 *   specifies a range that matches from {@code "a"} to {@code "z"} (inclusive).
 *   These forms can be mixed so [abce-g] matches {@code "a"}, {@code "b"}, 混合匹配
 *   {@code "c"}, {@code "e"}, {@code "f"} or {@code "g"}. If the character
 *   after the {@code [} is a {@code !} then it is used for negation so {@code  否定
 *   [!a-c]} matches any character except {@code "a"}, {@code "b"}, or {@code
 *   "c"}.
 *   <p> Within a bracket expression the {@code *}, {@code ?} and {@code \}
 *   characters match themselves. The ({@code -}) character matches itself if
 *   it is the first character within the brackets, or the first character
 *   after the {@code !} if negating.</p></li>
 *
 *   <li><p> The {@code { }} characters are a group of subpatterns, where  分组
 *   the group matches if any subpattern in the group matches. The {@code ","} 
 *   character is used to separate the subpatterns. Groups cannot be nested.
 *   </p></li>
 *
 *   <li><p> Leading period<tt>&#47;</tt>dot characters in file name are
 *   treated as regular characters in match operations. For example,
 *   the {@code "*"} glob pattern matches file name {@code ".login"}.
 *   The {@link Files#isHidden} method may be used to test whether a file
 *   is considered hidden.
 *   </p></li>
 *
 *   <li><p> All other characters match themselves in an implementation 其他字符匹配它们字节
 *   dependent manner. This includes characters representing any {@link 
 *   FileSystem#getSeparator name-separators}. </p></li>
 *
 *   <li><p> The matching of {@link Path#getRoot root} components is highly
 *   implementation-dependent and is not specified. </p></li>
 *
 * </ul>
 *
 * <p> When the syntax is "{@code regex}" then the pattern component is a
 * regular expression as defined by the {@link java.util.regex.Pattern}
 * class.
 *
 * <p>  For both the glob and regex syntaxes, the matching details, such as 对于glob和regex语法，匹配的细节
 * whether the matching is case sensitive, are implementation-dependent 是否区分大小写实现依赖没有指定
 * and therefore not specified.
 *
 * @param   syntaxAndPattern
 *          The syntax and pattern
 *
 * @return  A path matcher that may be used to match paths against the pattern
 *
 * @throws  IllegalArgumentException
 *          If the parameter does not take the form: {@code syntax:pattern}
 * @throws  java.util.regex.PatternSyntaxException
 *          If the pattern is invalid
 * @throws  UnsupportedOperationException
 *          If the pattern syntax is not known to the implementation
 *
 * @see Files#newDirectoryStream(Path,String)
 */
public abstract PathMatcher getPathMatcher(String syntaxAndPattern);

```
## 使用示例

- "*.java" 匹配文件名以".java"结尾

- "*.* 匹配包含点的文件名

- "*.{java,class}" 匹配文件以".java"或者".class"结尾

- "foo.?" 匹配文件名以"foo"开始和一个单个字符扩展名

- "/home/**" 匹配当前目录及其所有子目录

- "[xyz].txt" 匹配指定的单个字符

- "[a-c].txt" 匹配区间范围的单个字符串

- "[!a].txt" 匹配不是单个字符串

```java
/**
 * 以Glob模式匹配，目录
 * @param path
 * @param pattern
 * @return
 */
public static boolean globPathMatcher(String path, String pattern) {
    PathMatcher globPathMatcher = getPathMatcher(PathMatcherEnum.glob, pattern);
    return globPathMatcher.matches(Paths.get(path));
}

public static PathMatcher getPathMatcher(PathMatcherEnum pathMatcherEnum, String pattern) {
    return FileSystems.getDefault().getPathMatcher(pathMatcherEnum.name() + ":" + pattern);
}

//路径匹配枚举
enum PathMatcherEnum {
    glob, regex;
}

@Test
public void test() {

Assert.assertTrue(globPathMatcher("index.html", "*.{html,css}"));

Assert.assertTrue(globPathMatcher("/static/html/index.html", "**.{html,css}"));

}
```

