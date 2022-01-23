# JAVA做自定义正则替换


　　JAVA的正则替换用String类里的repalceAll方法就可以实现，但是这个方法有一个不小的缺陷，只能把正则查找出来的内容用同一段内容替换。现在我们要利用appendReplacement写一个以查找内容为参数进行自定义替换内容的方法。

<!-- more -->

　　先写代码，再解释。

# 1. 代码实现

## 1.1. 建立抽象类

``` java
package util.extregex;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
/**
 * 扩展替换抽象类
 * @author wjh
 *
 */
public abstract class AbstractExtReplace {
    
    private String regexstr = "";
    
    /**
     * 处理查找出来的数据
     * @param word 输入数据
     * @return 输出处理后的结果
     */
    protected abstract String treateWord(String word);
    
    /**
     * 继承方法里可以设置正则表达式
     * @param regexstr
     */
    protected void setRegexstr(String regexstr){
        this.regexstr = regexstr;
    }
    
    /**
     * 替换
     * @param input 输入内容
     * @return
     */
    public String extReplace(String input){
        return extReplace(regexstr, input, false);
    }
    
    /**
     * 替换
     * @param input 输入内容
     * @param useGroupName 是否用到groupname 默认false
     * @return
     */
    public String extReplace(String input, Boolean useGroupName){
        useGroupName = (useGroupName == null)?false:useGroupName;
        return extReplace(regexstr, input, useGroupName);
    }
    
    /**
     * 核心方法，正则替换
     * @param regex 输入正则表达式
     * @param input 输入查找原始内容
     * @param useGroupName 是否用到groupname
     * @return 返回替换后结果
     */
    protected String extReplace(String regexstr, String input, Boolean useGroupName){
         Pattern p = Pattern.compile(regexstr);
         Matcher m = p.matcher(input);
         StringBuffer sb = new StringBuffer();
         while (m.find()) {
             String replacement = treateWord(m.group());
             if(!useGroupName){
                 // 不用groupname的话这里需转义
                 replacement = replacement.replace("\\", "\\\\").replace("$", "\\$");
             }
             m.appendReplacement(sb, replacement);
         }
         m.appendTail(sb);
         return sb.toString();
    }
}
```

## 1.2. 举例实现

　　继承自AbstractExtReplace，可实现自定义替换

``` java
package util.extregex;
/**
 * 本类是将字符串中的大写字母转换为下划线加小写字母的形式 eg. myName 转为 my_name
 * @author wjh
 *
 */
public class LowerMethExtReplace extends AbstractExtReplace {
    /**
     * 将字符串中的大写字母转换为下划线加小写字母的形式 eg. myName 转为 my_name
     */
    public LowerMethExtReplace() {
        setRegexstr("[A-Z]");
    }
    @Override
    protected String treateWord(String word) {
        return "_" + word.toLowerCase();
    }
}
```

## 1.3. 测试

``` java
package main;
import util.extregex.LowerMethExtReplace;
public class Main {
    public static void main(String[] args) {
        AbstractExtReplace lowerMeth = new LowerMethExtReplace();
        String result = lowerMeth.extReplace("userName myValue");
        System.out.println(result);
        // 输出 user_name my_vamlue
    }
}
```

# 2. 说明

　　我们可以先看一下java的源码
　　String的replaceall方法：

``` java
public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}
```

　　可以看出String内用的replaceall就是用的Matcher中的replaceall
　　Matcher中的replaceall：

``` java
public String replaceAll(String replacement) {
    reset();
    boolean result = find();
    if (result) {
        StringBuffer sb = new StringBuffer();
        do {
            appendReplacement(sb, replacement);
            result = find();
        } while (result);
        appendTail(sb);
        return sb.toString();
    }
    return text.toString();
}
```

　　本文上面的方法其实就是扩展自这个方法。原方法中每次循环都默认赋值replacement，我们只需要把每次的赋值改成我们需要的即可，而正则查出的内容又都在matcher中，我们只需要把matcher中的数据作为appendReplacement的参数即可。
　　可以大概过一下appendReplacement方法。中间大部分的都是在处理groupname，等会儿会在后面写几个示例：

``` java
public Matcher appendReplacement(StringBuffer sb, String replacement) {
    // If no match, return error
    if (first < 0)
        throw new IllegalStateException("No match available");
    // Process substitution string to replace group references with groups
    int cursor = 0;
    StringBuilder result = new StringBuilder();
    while (cursor < replacement.length()) {
        char nextChar = replacement.charAt(cursor);
        if (nextChar == '\\') {
            cursor++;
            if (cursor == replacement.length())
                throw new IllegalArgumentException(
                    "character to be escaped is missing");
            nextChar = replacement.charAt(cursor);
            result.append(nextChar);
            cursor++;
        } else if (nextChar == '$') {
            // Skip past $
            cursor++;
            // Throw IAE if this "$" is the last character in replacement
            if (cursor == replacement.length())
               throw new IllegalArgumentException(
                    "Illegal group reference: group index is missing");
            nextChar = replacement.charAt(cursor);
            int refNum = -1;
            if (nextChar == '{') {
                cursor++;
                StringBuilder gsb = new StringBuilder();
                while (cursor < replacement.length()) {
                    nextChar = replacement.charAt(cursor);
                    if (ASCII.isLower(nextChar) ||
                        ASCII.isUpper(nextChar) ||
                        ASCII.isDigit(nextChar)) {
                        gsb.append(nextChar);
                        cursor++;
                    } else {
                        break;
                    }
                }
                if (gsb.length() == 0)
                    throw new IllegalArgumentException(
                        "named capturing group has 0 length name");
                if (nextChar != '}')
                    throw new IllegalArgumentException(
                        "named capturing group is missing trailing '}'");
                String gname = gsb.toString();
                if (ASCII.isDigit(gname.charAt(0)))
                    throw new IllegalArgumentException(
                        "capturing group name {" + gname +
                        "} starts with digit character");
                if (!parentPattern.namedGroups().containsKey(gname))
                    throw new IllegalArgumentException(
                        "No group with name {" + gname + "}");
                refNum = parentPattern.namedGroups().get(gname);
                cursor++;
            } else {
                // The first number is always a group
                refNum = (int)nextChar - '0';
                if ((refNum < 0)||(refNum > 9))
                    throw new IllegalArgumentException(
                        "Illegal group reference");
                cursor++;
                // Capture the largest legal group string
                boolean done = false;
                while (!done) {
                    if (cursor >= replacement.length()) {
                        break;
                    }
                    int nextDigit = replacement.charAt(cursor) - '0';
                    if ((nextDigit < 0)||(nextDigit > 9)) { // not a number
                        break;
                    }
                    int newRefNum = (refNum * 10) + nextDigit;
                    if (groupCount() < newRefNum) {
                        done = true;
                    } else {
                        refNum = newRefNum;
                        cursor++;
                    }
                }
            }
            // Append group
            if (start(refNum) != -1 && end(refNum) != -1)
                result.append(text, start(refNum), end(refNum));
        } else {
            result.append(nextChar);
            cursor++;
        }
    }
    // Append the intervening text
    sb.append(text, lastAppendPosition, first);
    // Append the match substitution
    sb.append(result);
    lastAppendPosition = last;
    return this;
}
```

　　我们可以看到java对groupname做了一些处理，如有$字符或\字符的话需要转义，这也是我在封装抽象类时添加了一个useGroupname参数的原因。这里的转义很有意思，共进行了2次转义。java中一次，替换时一次，也就是说如果准备替换成的字符串有两个反斜杠，那转义后就应该是4个反斜杠，如果写成String字符串明文表示的话还得进行一次转义，一共得写8个。

　　这里的逻辑也很明显只处理了能匹配最后一个值前面的部分，后面的却还没影儿呢。现在要做的是把尾巴接上，调用它的appendTail方法。

``` java
public StringBuffer appendTail(StringBuffer sb) {
    sb.append(text, lastAppendPosition, getTextLength());
    return sb;
}
```

　　到这里就基本结束了。

# 3. 附加

　　不知道最近是闲的无聊还是烦的无聊，多写一些。
　　Java中appendReplacement对groupname的处理，写几个示例大概就OK了吧。
　　还是拿Java官网的例子做改动，把cat改成 cat little cat 。英语语法肯定说不过去，不过反正我的英语老师也不会看，哈哈~

``` java
public static void main(String[] args) {
    Pattern p = Pattern.compile("(?<mycat>cat)");
    Matcher m = p.matcher("one cat two cats in the yard");
    StringBuffer sb = new StringBuffer();
    while (m.find()) {
        m.appendReplacement(sb, "${mycat} little ${mycat}");
    }
    m.appendTail(sb);
    System.out.println(sb.toString());
}
```
