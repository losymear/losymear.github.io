---
title: HTTPie使用指南
date: 2018-08-10 14:25:08
tags:
    - 命令行
---


`HTTPie`是一个在命令行中使用的HTTP客户端工具，目标是让CLI与web服务间的交互尽可能的人性化。它提供了一个简单的`http`命令，允许你使用简单而自然的语法发送任意的HTTP请求，并打印彩色的输出。`HTTPie`可以用于test、debug，通常用于与HTTP服务交互。
<!-- more -->
- [doc](https://httpie.org/doc)
- [github项目地址](https://github.com/jakubroztocil/httpie)
- [curl vs HTTPie](https://daniel.haxx.se/docs/curl-vs-httpie.html)

## Features
- 富有表现力和直观的语法
- 格式化的、彩色的终端输出
- 内置JSON支持
- 表单与文件上传
- HTTPS、代理、验证
- 任意请求数据
- 自定义header
- 持久化session
- 类`wget`的下载
- 支持Python2.6、2.7、3.x
- 支持Linux、Mac OSX、Windows
- 插件
- 文档
- 测试覆盖率

## Installation
- Mac环境使用`brew install httpie`。
- Linux环境使用对应的包管理器。 基于Debian的发行版使用`apt-get install httpie`，基于RPM的发行版本使用`yum install httpie`，arch linux使用`pacman -S httpie`。
- 所有环境可以通过`pip install --upgrade httpie`安装。
- 安装开发版本可以直接从GItHub下载，`pip install --upgrade https://github.com/jkbrzt/httpie/archive/master.tar.gz`。Mac OS可以用`brew install httpie --HEAD`安装。
虽然支持通过Python2.6、2.7安装`HTTPie`，不过推荐使用最新的Python3.x，以确保能使用较新的HTTP特性能够开箱即用，如`SNI`（Server Name Indication）。从`0.9.4`开始Homebrew默认使用Python3安装。要想查看使用的Python版本运行`http --debug`。

## 使用示例
基本使用方法为`http [flags] [METHOD] URL [ITEM [ITEM]]`。
可以运行`http --help`查看更多：
![http--help](_v_images/_httphelp_1533798884_873062724.png)
一些使用例子：
- 设置HTTP方法、请求头和JSON数据：`http PUT example.org X-API-Token:123 name=John`
- 表单提交：`http -f POST example.org hello=world`
- 设置输出的格式：`http -v example.org`
- 使用用户认证在这个[issue](https://github.com/jakubroztocil/httpie/issues/83)上发评论：`http -a USERNAME POST https://api.github.com/repos/jkbrzt/httpie/issues/83/comments body='HTTPie is awesome! :heart:'`
- 使用重定向来上传文件： `http example.org < file.json`
- 使用重定向来下载文件：`http example.org/file >file`
- 用类似`wget`的方式下载文件：`http --download example.org/file`
- 使用命名`session`来保持session：`http --session=logged-in -a username:password httpbin.org/get API-Key:123`、`http --session=logged-in httpbin.org/headers`
- 设置自定义的Host头部，来解决没有DNS记录的问题：`http localhost:8000 Host:example.com`

## HTTP方法
可以在请求URL前指定HTTP方法，如`http DELETE example.org/todos/7`，这发送了请求`DELETE /todos/7 HTTP/1.1`。
如果忽略了HTTP方法，那么默认会使用GET（没有请求数据时）或者POST（有请求数据时）。

## 请求URL
`HTTPie`发起请求需要一个URL。默认的scheme是`http://`，并且可以在参数中省略，比如`http example.org`就可以工作。

### Query参数
使用`param==value`的语法，可以让你不需要考虑`&`在终端的转义工作。此外，参数中的特殊字符也会自动转义（`HTTPie`希望URL已经被转义）。如果想在Google图片上搜索`HTTPie logo`，可以执行`http www.google.com search=='HTTPie logo' tbm==isch`，这实际发送了请求`GET /?search=HTTPie+logo&tbm=isch HTTP/1.1`。

### `localhost`的便捷写法
支持类似`curl`的对`localhost`的便捷写法。也就是说，`:3000`会扩展成`http://localhost:3000`。如果忽略端口，那么就会使用`80`端口。比如`http :`会扩展成`http localhost:80`。

### 设定默认的scheme
使用`--default-scheme <URL_SCHEME> `来设置默认的scheme。比如可以使用如下别名：
`alias https='http --default-scheme=https'`。

## 请求项
HTTPTie为各类请求项提供了方便的设置方式，比如指定header、简单的JSON与表单数据、文件、url参数。这些都是使用`key/value`的方式在URL之后指定，通过简单的分隔符：`:`、`=`、`:=`、`==`、`@`、`=@`、`:=@`。带`@`的表示值是一个文件路径。
|Item Type        |            Description        |
|--------                |---------------                |
|HTTP Header `Name:Value`|任意的HTTP 头部，如`X-API-Tken:123`|
|URL参数 `name==value`|把`key/value`对作为查询参数添加到URL末尾。|
|数据字段 `field=value`,`field=@file.txt`|请求的数据字段，默认会序列化为JSON对象，如果带有`--form`或`-f`则会转成表单字段|
|原始JSON字段，`filed:=json`,`field:=@file.json`|当JSON的字段需要是`Boolean`、`Number`、`Object`、`Array`时会很有用，比如`meals:='["ham","spam"]'`，或者`pies:=[1,2,3]`|
|表单的文件字段 `filed@/dir/file`|当带有`-f`或`--form`时有效。比如`screenshot@~/Pictures/img.png`。这会使请求是一个`multipart/form-data`请求|
注意数据字段不是唯一的指定请求参数的方法。输入重定向也可以传递任意请求参数。

### 转义规则
可以使用`\`来转义不希望作为分隔符的字符。比如`foo\==bar`表示`key/value`的数据字段（`foo=`和`bar`）而不是URL参数。
通常有必要使用引号， 如`foo='bar baz'`。
如果有数据字段名或者header以`-`开头，需要前置一个`--`，以免与`--arguments`混淆。比如`http httpbin.org/post  --  -name-starting-with-dash=foo -Unusual-Header:bar`表示的是：
```shell
POST /post HTTP/1.1
-Unusual-Header: bar
Content-Type: application/json

{
    "-name-starting-with-dash": "value"
}
```

## JSON
JSON是现代Web服务的能用格式，同时也是`HTTPie`默认的content type。
如果你的命令包含一些请求项，默认情况下它们会被序列化为JSON。`HTTPie`也会自动设置两个请求头——`Content-Type`为`application/json`，`Accept`为`application/json, */*`，都能被覆盖。

### 明确指定JSON
可以带上`--json`或`-j`来显示指定`Accept`为`application/json`（这是`http url Accept:'application/json, */*'`的简写）。同样，HTTPie会自动探测到响应的JSON数据，即使响应头的`Content-Type`是`text/plain`或者其他类型。

### 非字符串的JSON字段值
可以使用`:=`分隔符，这能让你使用原始类型数据来设置发送的JSON数据。同样文本或JSON文件也能使用`=@`与`:=@`来加入到发送的JSON数据。比如发起如下请求：
```shell
http PUT api.example.com/person/1 \
        name=John \
        age:=29 married:=false hobbies:='["http", "pies"]'  \          # Raw JSON
        description=@about-json.txt\        #会读取about-json.txt中的文本内容
        bookmarks:=@bookmarks.json
```
发送的请求数据如下：
```json
{
    "age": 29,
    "hobbies": [
        "http",
        "pies"
    ],
    "description": "John is a nice guy who likes pies.",
    "married": false,
    "name": "John",
    "bookmarks": {
        "HTTPie": "http://httpie.org",
    }
}
```
注意如果使用这一语法，在数据比较多时会显得笨拙，这时可以使用输入重定向：`http POST api.example.com/person/1 <person.json`

## 表单
提交表单与发送JSON十分类似，通常区别只在于添加`--form, -f`选项，这能确保数据字段被序列化为`application/x-www-form-urlencoded; charset=utf-8`，亦即请求的`Content-Type`。可以通过修改配置文件来将数据类型的默认格式设置为表单而不是JSON。

### 简单表单数据
![HTTPie form](_v_images/_httpieform_1533811389_1187608215.png)

### 文件上传表单
如果表单中有文件字段，那么序列化的类型为`multipart/form-data`。比如`http -f POST example.com/jobs name='John Smith' cv@~/Documents/cv.pdf`与如下HTML表单提交时传输的数据一样：
```html
<form enctype="multipart/form-data" method="post" action="http://example.com/jobs">
    <input type="text" name="name" />
    <input type="file" name="cv" />
</form>
```
注意`@`用于表示表单的文件字段，而`=@`则是将文件内的文本作为表单字段的值。

## HTTP header
要设置请求头，使用`Header:Value`表示法。比如`ttp example.org  User-Agent:Bacon/1.0  'Cookie:valued-visitor=yes;foo=bar'  \
    X-Foo:Bar  Referer:http://httpie.org/`设置了`Cookie`、`X-Foo`、`Referer`三个请求头。

### 默认值
有几个请求头的默认值：
```yml
GET / HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
User-Agent: HTTPie/<version>
Host: <从URL中获取>
```
除了`Host`外可以被覆盖。

### 空的header以及取消默认值
如果要取消指定的默认值可以使用`Header:`的方式，比如`http httpbin.org/headers Accept: User-Agent:`。
如果要发送空的请求头可以使用`Header;`的方式，比如`http httpbin.org/headers 'Header;'`。

## 认证
目前支持的认证方式有`Basic`和`Digest`，同时也能使用认证插件来支持更多方式。有两个命令行选项用于控制认证方式：
|选项        |            说明|
|--            |---------            |
|`--auth,-a`|传递一个`用户名:密码`对作为参数。或者如果只指定了用户名（`-a 用户名`），那么在请求发送前会提示你输入密码。要发送空的密码值使用`用户名:`。`用户名:密码@主机名`的方式也受支持（不过通过`-a`传参的方式优先级更高）|
|`--auth-type, --A`|指定认证机制。可选的值有`basic`和`digest`。默认值是`basic`所以一般可以省略|

### Basic认证
`http -a username:password example.org`

### Digest认证
`http -A digest -a username:password example.org`

### 密码输入提示
`http -a username example.org`

### `.netrc`
支持从`~/.netrc`文件中获取认证信息。
![.netrc](_v_images/_netrc_1533813145_1418254287.png)

### auth插件
可以通过安装插件来支持额外的认证机制。它们可以在[Python Package Index](https://pypi.python.org/pypi?%3Aaction=search&term=httpie&submit=search)中找到。以下是其中一部分：
- [httpie-api-auth](https://github.com/pd/httpie-api-auth) ApiAuth
- [httpie-aws-auth](https://github.com/jkbrzt/httpie-aws-auth) ApiAuth
- [httpie-edgegrid](https://github.com/akamai-open/httpie-edgegrid) EdgeGrid
- [httpie-oauth](https://github.com/jkbrzt/httpie-oauth) OAuth
- [requests-hawk](https://github.com/mozilla-services/requests-hawk) Hawk

## HTTP重定向
默认情况下，HTTP重定向不会进行跟踪，只会显示第一个响应：
`http httpbin.org/redirect/3`

### 跟踪`Location`
为了让`HTTPie`跟踪状态码为`30x`的HTTP响应的`Location`响应头，并返回最终的结果，使用`--follow,-F`选项：
`http --follow httpbin.org/redirect/3`

### 显示重定向中间过程的响应头
如果要显示中间过程的请求头/响应头，同时使用`--follow`和`--all`选项：
`http --follow --all httpbin.org/redirect/3`

### 限制最大重定向次数
默认重定向最多30次，可以使用`--max-redirects=<limit>`进行修改：
`http --follow --all --max-redirects=5 httpbin.org/redirect/3`

## 代理
可以通过`--proxy`来指定为每个协议使用的代理（参数中包含协议，防止跨协议的重定向）：
`http --proxy=http:http://10.10.1.10:3128 --proxy=https:https://10.10.1.10:1080 example.org`。
使用Basic认证：
`http --proxy=http:http://user:pass@10.10.1.10:3128 example.org`。

### 环境变量
可以使用环境变量`HTTP_PROXY`和`HTTPS_PROXY`来配置代理，底层的`Requests`库会使用它们。如果想对特定的host禁用代理，可以使用`NO_PROXY`环境变量。
在`~/.bash_profile`中进行如下配置：
```rc
export HTTP_PROXY=http://10.10.1.10:3128
export HTTPS_PROXY=https://10.10.1.10:1080
export NO_PROXY=localhost,example.com
```

### SOCKS
要想使用SOCKS代理需要安装`requests[socks]`，通过执行`pip install -U requests[socks]`。
使用方式也相似：`http --proxy=http:socks5://user:pass@host:port --proxy=https:socks5://user:pass@host:port example.org`。

## HTTPS
### 服务器SSL证书认证
如果要跳过主机的SSL证书认证，可以使用`--verify=no`（默认为`yes`）：
`http --verify=no https://example.org`。

### 自定义CA bundle
可以通过`--verify=<CA_BUNDLE_PATH>`来设置一个自定义的CA bundle路径：
`http --verify=/ssl/custom_ca_bundle https://example.org`。

### 客户端SSL证书
如果要使用客户端证书来进行SSL通信，可以通过使用`--cert`来设置：
`http --cert=client.pem https://example.org`。
如果证书文件中不包含私钥可以使用`--cert-key`来设置：
`http --cert=client.crt --cert-key=client.key https://example.org`。

### SSL版本
使用`--ssl=<PROTOCOL>`来指定要使用版本。默认为`SSL v2.3`，它会协商服务器和你本地安装的OpenSSL所支持的最高版本。可用的协议有`ssl2.3`、`ssl3`、`tls1`、`tls1.1`、`tls1.2`（实际可用的版本可能出处，取决于你安装的OpenSSL）。
使用脆弱的SSL v3来与过时的服务器通信：
`http --ssl=ssl3 https://vulnerable.example.org`

### `SNI`（Server Name Indication）
如果你的`HTTPie`使用的Python版本低于2.7.9，并且想要和使用`SNI`的服务器通信，那么就需要安装额外的依赖：
`pip install --upgrade requests[security]`
你可以使用如下命令来查看是否支持`SNI`：
`http https://sni.velox.ch`

## 输出选项
`HTTPie`默认只打印最终的响应结果，并打印出整个响应信息（包括响应头和响应体）。你可以使用下列选项来控制打印哪些信息：
- `--headers, -h`    只打印响应头
- `--body, -b`    只打印响应体
- `--verbose, -v`    打印全部的HTTP交换信息（请求和响应）。这个选项同时会激活`--all`
- `--print, -p`    选择要打印的HTTP交换信息的哪些部分

`--verbose`选项在调试或者生成文档时很有用。

### 选择要打印的HTTP交换信息
其他的输出选项其实都是使用的`--print`。它接受一个字符串，每个字符表示HTTP交换信息的一部分：
|    字符        |对应的HTTP交换信息部分        |
|----------        |---------------------        |
|`H`        |请求头|
`B`    | 请求体    |
|`h`|    响应头|
|`b`|    响应体|

比如要打印请求和响应头可以执行`http --print=Hh PUT httpbin.org/put hello=world`。

### 查看中间过程的请求/响应信息
如果要查看所有的HTTP通信，即最终结果与可能的中间过程中的信息，可以使用`--all`选项。“中间过程信息”包括重定向跟踪（要开启`--follow`选项），及使用`digest`认证方式时的第一次未认证请求，等等。
例如，查看重定向：`http --all --follow httpbin.org/redirect/3`
中间过程的请求响应信息默认使用`--print, -p`以及上述使用它的其他选项（`-h`、`-b`、`-v`、`-p`）。如果不想这样，可以使用`--history-print, -P`选项，它使用与`--print, -p`相同的选项，不过只对中间过程生效。
`http -A digest -a foo:bar --all -p Hh -P H httpbin.org/digest-auth/auth/foo/bar`

### 有选择的响应体获取
为了进行优化，响应体只有当它只输出的一部分时才会从服务端下载下来。这类似于发起一个`HEAD`请求，不过它对你使用的所有HTTP请求有效。
假如有一个API，它会在资源更新时返回整个资源，不过你只对响应头有兴趣，因为你只需要知道响应头来查看资源更新后返回的状态码。
```sh
http --headers PATCH example.org/Really-Huge-Resource name='New Name'`
```
只为上述命令只打印HTTP头部信息，那么当收到全部的响应头信息后与服务端间的连接就会关闭。这样，你就不会因为不感兴趣的响应体而浪费时间与带宽。响应头总会获取，即使你不打印它。

## 输入重定向
一种通用的传递数据的方法是使用管道（`pipe`）来将数据重定向到`stdin`。这些数据会作为请求体发送。有多种有效的使用`pipe`的方式：
从一个文件中重定向：
```sh
http PUT example.com/person/1 X-API-Token:123 < person.json
```
或者从另外一个文件的输出中：
```sh
grep '401 Unauthorized' /var/log/httpd/error_log | http POST example.org/intruders
```
对于一些简单的数据可以使用`echo`：
```sh
echoecho  '{"name": "John"}''{"name": "John"}  | http PATCH example.com/person/1 X-API-Token:123
```
甚至可以把多个web请求连接起来：
```sh
http GET https://api.github.com/repos/jkbrzt/httpie | http POST httpbin.org/post
```
你可以使用`cat`来在终端输入多行数据：
```sh
cat | http POST example.com
<paste>
^D
```
在OS X系统中你可以使用`pbpaste`来使用剪贴板中的数据：
```sh
pbpaste | http PUT example.com
```
通过`stdin`来传递数据不能与在命令行中指定数据的方式混用：
```sh
echo 'data' | http POST example.org more=data   # 这不种使用方式无效
```
如果不想让`HTTPie`从`stdin`中读取数据可以使用`--ignore-stdin`选项。

### 从文件中读取数据
一个使用`stdin`的替代方法是指定一个文件名（如`@/path/to/file`），文件内容会被使用，如同从`stdin`中获取的一样。
这种方式有一个好处，即`Content-Type`会根据文件扩展名来进行合适的设置。比如下列请求会从XML文件中获取内容发送，请求头会自动设置为`Content-Type: application/xml`：
```sh
http PUT httpbin.org/put @/data/file.xml
```

## 终端输出
`HTTPie`默认做了几件事件，来使输出的内容容易阅读：

### 高亮与格式化
HTTP头部信息与body信息都会被语法高亮（如果body的数据能够被高亮的话）。如果你不希望使用默认的选项，你可以使用`--style`选项来选择合适的值（执行`http --help`来查看可选的值）。
同时会进行如下格式化：
- HTTP头部信息会进行排序
- JSON数据会被缩进、会按key排序、unicode转义的字符会被转换为它代表的字符

可以使用`--pretty`来进行控制。`--pretty=colors`表示只应用高亮；使用`--pretty=format`表示只应用格式化。使用`--pretty=all`表示应用两者（默认值），使用`--pretty=none`关闭输出（使用输出重定向时的默认值）。

### 二进制数据
安全考虑，二进制的返回数据在终端会被抑制输出。在`--pretty`的输出重定向中二进制数据也会被抑制。`HTTPie`在发现返回数据是二进制数据时会立即关闭连接。
响应为二进制数据时会提示：`NOTE: binary data not shown in terminal`。
![http binary](_v_images/_httpbinary_1533868650_1952083146.png)

### 重定向输出
`HTTPie`对重定向输出有和终端输出不一样的默认设定：
- 不会应用高亮及格式化，除非指定了`--pretty`
- 只打印响应体，除非用`-h`，`-v`，`-p`，`-b`进行设置
- 二进制数据不被抑制
这些设定的原因是，将`HTTPie`的输出pipe到别的程序或者下载到文件中时不需要进行额外的设定。大多数情况下，当重定向输出时，只有原始的响应信息才有意义。
下载文件：
```sh
http example.org/Movie.mov > Movie.mov
```
下载一个Octocat的图片，使用`ImageMagick`修改尺寸，上传到某个地方：
```sh
http octodex.github.com/images/original.jpg | convert - -resize 25% -  | http example.org/Octocats
```
强制使用高亮和格式化，并在`less`中显示请求和响应：
```sh
http octodex.github.com/images/original.jpg | convert - -resize 25% -  | http example.org/Octocats
```
其中的`-R`选项表示让`less`解释转义字符，来显示高亮。
你可以在终端配置中添加如下函数，来方便的显示高亮的响应头和响应体（添加到`~/.bash_profile`）：
```sh
function httpless {
    # `httpless example.org'
    http --pretty=all --print=hb "$@" | less -R;
}
```

## 下载模式
`HTTPie`提供了一个下载模式，在行为上类似`wget`。
当使用`--download, -d`选项时，响应头会在终端打印，响应体会保存到文件中，并会有一个进度条指示当前进度。

### 下载文件名
如果没有提供`--output, -o`选项，输出的文件名会根据`Content-Disposition`（如果有的话），或者根据URL和`Content-Type`。如果已经有同名文件存在，`HTTPie`会添加一个后缀。比如如果文件如为`pie1.txt`且文件夹已经有同名文件，会将响应体保存到`pie1.txt-1`中。

### 下载同时pipe
你可以将响应体重定向到其他程序的同时，显示响应头和进度条。
```sh
http -d https://github.com/jkbrzt/httpie/archive/master.tar.gz |  tar zxf -
```

### 恢复下载
如果指定了`--output, -o`，你可以通过一个`--continue, -c`选项来恢复一个部分下载。这只有在服务器支持`Range`请求和`206 Partial Contet`响应时才有效。如果不支持，那么就会下载整个文件：
```sh
http -dco file.zip example.org/file
```

### 其他说明
- `--download`只是改变了响应体会被如何处理。
- 仍然可以自定义请求头、使用session、`--verbose`等。
- `--download`会跟踪重定向（`--follow`）。
- 如果文件没有被完全下载下来，`HTTPie`会以状态码1退出。
- `Accept-Encoding`不能与`--download`同时使用。


## 流式响应
响应能够下载并以chunks打印到终端（允许在不占用过多的内存前提下进行流式传输并下载大文件）。但如果使用了高亮和格式化，那么整个响应会先被缓存，然后一次性处理。

### 禁用缓存
你可以使用`--stream, -S`选项来做两件事：
- 输出会以较小和chunk来flush，而不使用buffer。这会让`HTTPie`的行为看起来像`tail -f`
- 流式被启用，即使响应被高亮和格式化：它会被用于响应的每一行，然后立即flush。这可以让一个长连接有较好看的输出，比如用于Twitter的流式API时。

### Example
美化过的流式响应：
```sh
http --stream -f -a YOUR-TWITTER-NAME https://stream.twitter.com/1/statuses/filter.json track='Justin Bieber'
```
类似`tail -f`的美化过的流式输出：
```sh
# Send each new tweet (JSON object) mentioning "Apple" to another#
# server as soon as it arrives from the Twitter streaming API:
http --stream -f -a YOUR-TWITTER-NAME https://stream.twitter.com/1/statuses/filter.json track=Apple \
| while read tweet; do echo "$tweet" | http POST example.org/tweets ; done
```

## Session（会话）
默认情况下，`HTTPie`的每一个请求都和之前的请求完全独立。
不过`HTTPie`支持持久化的session，通过使用`--session=SESSION_NAME_OR_PATH`选项。在一个会话中，自定义的header（除了以`Content-`和`If-`开头的）、认证和cookie（手动设定的或由服务器发送的）会在对同一个host发请求时保持一致。
```sh
# Create a new session
http --session=/tmp/session.json example.org API-Token:123

# Re-use an existing session — API-Token will be set:
http --session=/tmp/session.json example.org
```
所有的会话信息，包括认证、cookie、自定义头都会以纯文本保存。这意味着会话文件能手动地创建和修改——它们就是普通的JSON文件。

### 命名session
你可以为一个host创建一个或多个命令session。比如，你可以使用如下命令来为`example.org`创建一个名为`user1`的session：
```sh
http --session=user1 -a user1:password example.org X-Foo:Bar
```
之后，你可以直接用`user`来使用这个session。指定后，先前使用的认证与HTTP头都会自动设定：
```sh
http --session=user1 example.org
```
如果想使用新的session，可以指定另外一个名字：
```sh
http --session=user2 -a user2:password example.org X-Bar:Foo
```
命名session保存在`~/.httpie/sessions/<host>/<name>.json`文件夹中（Windows上是在`%APPDATA%\httpie\sessions\<host>\<name>.json`）。

### 匿名session
你可以不命名，而直接指定session文件路径。这能让你的session在不同的host间复用：
```sh
http --session=/tmp/session.json example.org
http --session=/tmp/session.json admin.example.org
http --session=~/.httpie/sessions/another.example.org/test.json example.org
http --session-read-only=/tmp/session.json example.org
```

### 只读session
如果想一个session在被创建后不被更改，可以使用`--session-read-only=SESSION_NAME_OR_PATH`。


## 配置
`HTTPie`使用一个简单的JSON配置文件。

### 配置文件路径
默认的配置文件路径为`~/.httpiie/config.json`（Windows是在`%APPDATA/httpie/config.json`）。可以通过`HTTPIE_CONFIG_DIR`环境变量来修改配置文件所在的文件夹。

### 配置项
- `default_options`    一个默认选项的数组，应用于每一个`HTTPie`请求。比如如果你想要修改默认的高亮风格和输出选项，可以`"default_options": ["--style=fruity", "--body"]`。另外一个有用的选项是`"--session=default"`，这会让所有的`HTTPie`请求都使用session。或者你也可以将默认的content type从JSON改为表单，通过`--form`选项。
- `__meta__`    `HTTPie`自动将它的元信息保存到`__meta__`中。不要修改它。

### 不使用默认配置
如果想对特定的请求不使用默认配置，可以使用`-0no-OPTION`，比如`--no-stype`、`--no-session`。

## 脚本
对于在shell脚本中使用`HTTPie`，有一个非常有用的选项`--check-status`。它会在状态码是`3xx`、`4xx`、`5xx`时让`HTTPie`退出，并带有错误码。错误码可以是`3`（除非设置`--follow`）、`4`或`5`。
```sh
#!/bin/bash

if http --check-status --ignore-stdin --timeout=2.5 HEAD example.org/health &> /dev/null; then
    echo 'OK!'
else
    case $? in
        2) echo 'Request timed out!' ;;
        3) echo 'Unexpected HTTP 3xx Redirection!' ;;
        4) echo 'HTTP 4xx Client Error!' ;;
        5) echo 'HTTP 5xx Server Error!' ;;
        6) echo 'Exceeded --max-redirects=<n> redirects!' ;;
        *) echo 'Other Error!' ;;
    esac
fi
```

### 最佳实践
除非是交互式的调用，最后不要从`stdin`中读取。因此你可能需要设置`--ignore-stin`的默认选项。
如果没有这个选项，`HTTPie`很可能会挂起。可能发生的原因是如果`HTTPie`是从一个定时任务中调用，`stdin`并没有关联一个终端。此时适用“重定向输入”规则，即`HTTPie`会期望请求体被输入，但是因为不存在数据或`EOF`，因此就会停止。因此除非你想将一些数据pipe到`HTTPie`，否则最好要设置`--ignore-stin`。
当然，你也可以不使用默认的30s的超时时间，而是通过`--timeout`来设置对你来说合适的值。



----
版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](https://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
