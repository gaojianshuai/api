# RPC API签名 {#concept_vbl_2p3_ndb .concept}

为保证API的安全调用，在调用API时阿里云会对每个API请求通过签名（Signature）进行身份验证。无论使用HTTP还是HTTPS协议提交请求，都需要在请求中包含签名信息。

**说明：** 阿里云提供了多种语言的SDK及第三方SDK，可以免手动签名编码的麻烦。单击[这里](https://develop.aliyun.com/tools/sdk)了解更多阿里云SDK的信息。

## 概述 {#section_vwl_wq3_ndb .section}

RPC API要按如下格式在API请求的Query中增加签名（Signature）：

```
https://Endpoint/?SignatureVersion=1.0&SignatureMethod=HMAC-SHA1&Signature=CT9X0VtwR86fNWSnsc6v8YGOjuE%3D&SignatureNonce=3ee8c1b8-83d3-44af-a94f-4e0ad82fd6cf
```

其中：

-   SignatureMethod：签名方式，目前支持HMAC-SHA1。
-   SignatureVersion：签名算法版本，目前版本是 1.0。
-   SignatureNonce：唯一随机数，用于防止网络重放攻击。用户在不同请求间要使用不同的随机数值,建议使用通用唯一识别码（Universally Unique Identifier, UUID）。
-   Signature: 使用AccessKey Secret对请求进行对称加密后生成的签名。

## 计算签名 {#section_ysb_jr3_ndb .section}

签名算法遵循RFC 2104 HMAC-SHA1规范，使用AccessSecret对编码、排序后的整个请求串计算HMAC值作为签名。签名的元素是请求自身的一些参数，由于每个API请求内容不同，所以签名的结果也不尽相同。

```
Signature = Base64( HMAC-SHA1( AccessSecret, UTF-8-Encoding-Of(
StringToSign)) )
```

完成以下操作，计算签名：

1.  **构建待签名字符串**
    1.  使用请求参数构造规范化的请求字符串（Canonicalized Query String）:
        1.  按照参数名称的字典顺序对请求中所有的请求参数（包括公共请求参数和接口的自定义参数，但不包括公共请求参数中的Signature参数）进行排序。

            当使用GET方法提交请求时，这些参数就是请求URI中的参数部分，即URI中“?”之后由“&”连接的部分。

        2.  对排序之后的请求参数的名称和值分别用UTF-8字符集进行URL编码。编码规则如下：
            -   对A-Z、a-z和0-9以及“-”、“\_”、“.”和“~”不编码。

            -   其它字符编码成 `%XY` 的格式，其中 `XY` 是字符对应ASCII码的16进制表示。比如英文的双引号（””）对应的编码为 `%22`。

            -   扩展的UTF-8字符，编码成 `%XY%ZA…`的格式。

            -   英文空格要编码成 `%20`，而不是加号（+）。

                该编码方式和一般采用的 `application/x-www-form-urlencoded` MIME格式编码算法（比如 Java标准库中的 `java.net.URLEncoder`的实现）存在区别。编码时可以先用标准库的方式进行编码，然后把编码后的字符串中的加号（+）替换成 `%20`，星号（\*）替换成 `%2A`，`%7E`替换回波浪号（~），即可得到上述规则描述的编码字符串。本算法可以用下面的`percentEncode`方法来实现：

                ```
                private static final String ENCODING = "UTF-8";
                private static String percentEncode(String value) throws UnsupportedEncodingException 
                {
                return value != null ? URLEncoder.encode(value, ENCODING).replace("+", "%20").replace("*", "%2A").replace("%7E", "~") : null;
                }
                ```

            -   将编码后的参数名称和值用英文等号（=）进行连接。
            -   将等号连接得到的参数组合按步骤 i 排好的顺序依次使用“&”符号连接，即得到规范化请求字符串。
    2.  将第一步构造的规范化字符串按照下面的规则构造成待签名的字符串。

        ```
        StringToSign=
              HTTPMethod + “&” +
              percentEncode(“/”) + ”&” +
               percentEncode(CanonicalizedQueryString)
        ```

        其中：

        -   HTTPMethod是提交请求用的HTTP方法，比如GET。
        -   percentEncode\(“/”\)是按照步骤1.1中描述的 URL 编码规则对字符 “/” 进行编码得到的值，即 %2F。
        -   percentEncode\(CanonicalizedQueryString\)是对步骤1中构造的规范化请求字符串按步骤 1.2 中描述的URL编码规则编码后得到的字符串。
2.  **计算签名**
    1.  按照RFC2104的定义，计算待签名字符串（StringToSign）的HMAC值。

        **说明：** 计算签名时使用的Key就是您持有的AccessKey Secret并加上一个 “&” 字符（ASCII:38）, 使用的哈希算法是SHA1。

    2.  按照Base64编码规则把上面的HMAC值编码成字符串，即得到签名值（Signature）。
    3.  将得到的签名值作为Signature参数添加到请求参数中。

        **说明：** 得到的签名值在作为最后的请求参数值提交时要和其它参数一样，按照[RFC3986](https://tools.ietf.org/html/rfc3986)的规则进行URL编码。


## 示例 {#section_hlb_bt3_ndb .section}

以DescribeRegionsAPI 为例，假设使用的`AccessKey Id` 为 `testid`， `AccessKey Secret`为`testsecret`。 签名前的请求URL如下：

```
http://ecs.aliyuncs.com/?Timestamp=2016-02-23T12:46:24Z&Format=XML&AccessKeyId=testid&Action=DescribeRegions&SignatureMethod=HMAC-SHA1&SignatureNonce=3ee8c1b8-83d3-44af-a94f-4e0ad82fd6cf&Version=2014-05-26&SignatureVersion=1.0
```

使用`testsecret&`，计算得到的签名值是：

```
OLeaidS1JvxuMvnyHOwuJ+uX5qY=
```

最后将签名作为Signature参数加入到URL请求中，最后得到的URL为：

```
http://ecs.aliyuncs.com/?SignatureVersion=1.0&Action=DescribeRegions&Format=XML&SignatureNonce=3ee8c1b8-83d3-44af-a94f-4e0ad82fd6cf&Version=2014-05-26&AccessKeyId=testid&Signature=OLeaidS1JvxuMvnyHOwuJ+uX5qY=&SignatureMethod=HMAC-SHA1&Timestamp=2016-02-23T12%3A46%3A24Z
```

