# HTTP content-type

Content-Type（内容类型），一般是指网页中存在的 Content-Type，用于定义网络文件的类型和网页的编码，决定浏览器将以什么形式、什么编码读取这个文件，这就是经常看到一些 PHP 网页点击的结果却是下载一个文件或一张图片的原因。

Content-Type 标头告诉客户端实际返回的内容的内容类型。

语法格式：

```
Content-Type: text/html; charset=utf-8
Content-Type: multipart/form-data; boundary=something
```

# 命名格式

一个MIME类型包括一个类型（type），一个子类型（subtype）。此外可以加上一个或多个可选参数（optional parameter）。其格式为

`类型名 / 子类型名 [ ; 可选参数 ]`

目前已被注册的类型名有application、audio、example、image、message、model、multipart、text，以及video。chemical是一个非官方的常用类型名。此外，非标准的类型名一般会加上x-前缀，但这种做法已经过时。

子类型名通常是一个媒体形式被冠以的名称，不过子类型名中也会有其它信息，包括厂商信息、产品信息、分类信息（子类型会被归进一个树状的分类结构中）、后缀等等。树结构分类信息以被.相互连接的字符串表示。每一个由.分隔开的部分又可以加上与其以-相连接的附加信息。此外，子类型名中也会有放在最后，与前面的内容以 + 相连接的后缀。因此，一个媒体类型的格式可以被更加细地表示为：

`类型名 / [ 树结构分类信息（中间可能有一个或多个“.”） ] 子类型名（中间可能有一个或多个“-”） [ + 后缀 ] [ ; 可选参数 ]`

这些信息遵循注册树（见下）的规定。

# 媒体类型列表

IANA 维护着一个媒体类型和字符编码的记录列表。他们的列表通过互联网向公众开放。

## Type application

分别对于不同用途的文件：

```
application/atom+xml：Atom feeds
application/ecmascript：ECMAScript/JavaScript;[4]（相当于application/javascript但是严格的处理规则）
application/EDI-X12：EDI ANSI ASC X12资料[5]
application/EDIFACT：EDI EDIFACT资料[5]
application/json：JSON（JavaScript Object Notation）[6]
application/javascript：ECMAScript/JavaScript[4]（相当于application/ecmascript但是宽松的处理规则）它不被IE 8或更早之前的版本所支持。虽然可以改用text/javascript，但它却被RFC 4329定义为过时。在HTML5之中，<script>标签的type的属性是可省略的，因为所有的浏览器即使在HTML5以前都一直默认使用JavaScript。
application/octet-stream:任意的二进制文件（通常做为通知浏览器下载文件）
application/ogg：Ogg, 视频文件格式[9]
application/pdf：PDF（Portable Document Format）[10]
application/postscript：PostScript[7]
application/rdf+xml：Resource Description Framework[11]
application/rss+xml：RSS feeds
application/soap+xml：SOAP
application/font-woff：Web Open Font Format;（推荐使用；使用application/x-font-woff直到它变为官方标准）
application/xhtml+xml：XHTML
application/xml：XML文件
application/xml-dtd：DTD文件
application/xop+xml：XML-binary Optimized Packaging[15]
application/zip：ZIP压缩包
application/gzip：Gzip
```

## Type audio

数字音频文件：

```
audio/mp4：MP4音频档案
audio/mpeg：MP3或其他MPEG音频档案
audio/ogg：Ogg音频档案
audio/vorbis：Vorbis音频档案
audio/vnd.rn-realaudio：RealAudio音频档案
audio/vnd.wave：WAV音频档案
audio/webm：WebM音频档案
audio/flac：FLAC音频档案
```

## Type image

```
image/gif：GIF图像文件[23]
image/jpeg：JPEG图像文件[23]
image/png： PNG图像文件[24]
image/webp： WebP图像文件
image/svg+xml：SVG向量图像文件[25]
image/tiff：TIFF图像文件[26]
image/icon：ICO图片文件。
```

# Type model

三维计算机图形文件：

```
model/example[27]
model/iges：IGS files, IGES files[28]
model/mesh：MSH files, MESH files[28]
model/vrml：WRL files, VRML files[28]
model/x3d+binary：X3D ISO standard for representing 3D computer graphics, X3DB binary files
model/x3d+vrml：X3D ISO standard for representing 3D computer graphics, X3DV VRML files
model/x3d+xml：X3D ISO standard for representing 3D computer graphics, X3D XML files
```

# Type multipart

```
multipart/form-data ： 需要在表单中进行文件上传时，就需要使用该格式
```

# Type text

```
text/css：CSS文件[29]
text/csv：CSV文件[30]
text/html：HTML文件[31]
text/javascript (过时): JavaScript; 在 RFC 4329 中定义并舍弃，以减少使用，推荐使用 application/javascript。然而，相比于 application/javascript ，在 HTML 4 和 5 中，可以使用text/javascript ，且有跨浏览器的支持。因为在使用 <script> 时，对于其 "type" 属性 ，所有浏览器都会使用正确的默认值（尽管 HTML 4 的规格中明确要求），所以 HTML 5 中定义为选择性的，且没必要。
text/plain:纯文字内容[32]
text/vcard：vCard（电子名片）[33]
text/xml：XML[14]
```

# Type video

视频文件格式文件（可能包含数字视频与数字音频）：

```
video/mpeg：MPEG-1视频文件
video/mp4：MP4视频文件
video/ogg：Ogg视频文件
video/quicktime：QuickTime视频文件
video/webm：WebM视频文件（基于Matroska基础）
video/x-matroska：Matroska（多媒体封装格式）
video/x-ms-wmv：Windows Media Video视频文件
video/x-flv：Flash Video（FLV档）
```

| 文件后缀 | MIME TYPE |
| ------------------------------------------------------------ | ---- |
| .doc | application/msword |
|.dot | application/msword |
|.docx | application/vnd.openxmlformats-officedocument.wordprocessingml.document |
| .dotx | application/vnd.openxmlformats-officedocument.wordprocessingml.template |
|.docm | application/vnd.ms-word.document.macroEnabled.12 |
| .dotm | application/vnd.ms-word.template.macroEnabled.12 |
|.xls | application/vnd.ms-excel |
|.xlt | application/vnd.ms-excel | 
| .xla | application/vnd.ms-excel | 
|.xlsx | application/vnd.openxmlformats-officedocument.spreadsheetml.sheet|
|.xltx | application/vnd.openxmlformats-officedocument.spreadsheetml.template |  
| .xlsm| application/vnd.ms-excel.sheet.macroEnabled.12 | 
| .xltm |application/vnd.ms-excel.template.macroEnabled.12 | 
| .xlam |application/vnd.ms-excel.addin.macroEnabled.12 | 
| .xlsb |application/vnd.ms-excel.sheet.binary.macroEnabled.12 | 
| .ppt |application/vnd.ms-powerpoint | 
| .pot |application/vnd.ms-powerpoint | 
| .pps |application/vnd.ms-powerpoint | 
| .ppa |application/vnd.ms-powerpoint | 
| .pptx |application/vnd.openxmlformats-officedocument.presentationml.presentation | 
| .potx |application/vnd.openxmlformats-officedocument.presentationml.template | 
| .ppsx |application/vnd.openxmlformats-officedocument.presentationml.slideshow | 
| .ppam |application/vnd.ms-powerpoint.addin.macroEnabled.12 |
| .pptm |application/vnd.ms-powerpoint.presentation.macroEnabled.12 | 
|.potm |application/vnd.ms-powerpoint.presentation.macroEnabled.12 |
| .ppsm |application/vnd.ms-powerpoint.slideshow.macroEnabled.12 | 
|.zip | application/zip | 
|.tar| application/x-tar |



# 参考

- https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E5%AA%92%E4%BD%93%E7%B1%BB%E5%9E%8B