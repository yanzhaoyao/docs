# APP接口开发规范

### 摘要

本规范将对开发流程、常见问题的解决方法、各角色的职责进行详细描述。

### http协议

客户端http协议使用注意点：

#### 请求头

- Accept

```
Accept: */*
```

不应使用指定accept值，在部分网络返回406。

#### 响应头

- Content-Type

```
Content-Type: application/json;charset=UTF-8
```

- Cache-Control

```
Pragma:no-cache
Cache-Control:no-cache
Date:Tue, 13 Jun 2017 06:20:57 GMT
Last-Modified:Tue, 13 Jun 2017 06:20:57 GMT
Expires:Thu, 01 Jan 1970 00:00:00 GMT
```

- Accept-Charset

SpringMVC 3.1 版本要特别注意，设置writeAcceptCharset = false 否则会出现如下超长的header

```xml
Accept-Charset: big5, big5-hkscs, compound_text, euc-jp, euc-kr, gb18030, gb2312, gbk, ibm-thai, ibm00858, ibm01140, ibm01141, ibm01142, ibm01143, ibm01144, ibm01145, ibm01146, ibm01147, ibm01148, ibm01149, ibm037, ibm1026, ibm1047, ibm273, ibm277, ibm278, ibm280, ibm284, ibm285, ibm297, ibm420, ibm424, ibm437, ibm500, ibm775, ibm850, ibm852, ibm855, ibm857, ibm860, ibm861, ibm862, ibm863, ibm864, ibm865, ibm866, ibm868, ibm869, ibm870, ibm871, ibm918, iso-2022-cn, iso-2022-jp, iso-2022-jp-2, iso-2022-kr, iso-8859-1, iso-8859-13, iso-8859-15, iso-8859-2, iso-8859-3, iso-8859-4, iso-8859-5, iso-8859-6, iso-8859-7, iso-8859-8, iso-8859-9, jis_x0201, jis_x0212-1990, koi8-r, koi8-u, shift_jis, tis-620, us-ascii, utf-16, utf-16be, utf-16le, utf-32, utf-32be, utf-32le, utf-8, windows-1250, windows-1251, windows-1252, windows-1253, windows-1254, windows-1255, windows-1256, windows-1257, windows-1258, windows-31j, x-big5-hkscs-2001, x-big5-solaris, x-euc-jp-linux, x-euc-tw, x-eucjp-open, x-ibm1006, x-ibm1025, x-ibm1046, x-ibm1097, x-ibm1098, x-ibm1112, x-ibm1122, x-ibm1123, x-ibm1124, x-ibm1364, x-ibm1381, x-ibm1383, x-ibm33722, x-ibm737, x-ibm833, x-ibm834, x-ibm856, x-ibm874, x-ibm875, x-ibm921, x-ibm922, x-ibm930, x-ibm933, x-ibm935, x-ibm937, x-ibm939, x-ibm942, x-ibm942c, x-ibm943, x-ibm943c, x-ibm948, x-ibm949, x-ibm949c, x-ibm950, x-ibm964, x-ibm970, x-iscii91, x-iso-2022-cn-cns, x-iso-2022-cn-gb, x-iso-8859-11, x-jis0208, x-jisautodetect, x-johab, x-macarabic, x-maccentraleurope, x-maccroatian, x-maccyrillic, x-macdingbat, x-macgreek, x-machebrew, x-maciceland, x-macroman, x-macromania, x-macsymbol, x-macthai, x-macturkish, x-macukraine, x-ms932_0213, x-ms950-hkscs, x-ms950-hkscs-xp, x-mswin-936, x-pck, x-sjis_0213, x-utf-16le-bom, x-utf-32be-bom, x-utf-32le-bom, x-windows-50220, x-windows-50221, x-windows-874, x-windows-949, x-windows-950, x-windows-iso2022jp
```

```xml
<bean id="strHttpMessageConverter"
    class="org.springframework.http.converter.StringHttpMessageConverter">
    <property name="supportedMediaTypes">
      <util:list>
        <value>text/plain;charset=UTF-8</value>
        <value>application/json;charset=UTF-8</value>
      </util:list>
    </property>
    <property name="writeAcceptCharset" value="false"></property>
  </bean>
```

### 接口定义

#### 原则

根据MVC/MVVM模型，接口返回数据是model，客户端打开的界面是view。

#### 接口职责划分

接口以客户端UI为准，返回当前view需要的显示的数据。在减少请求数的同时，减少服务端返回的数据量。
 需要二次点击展示的数据，点击后单独请求，减少一次返回的数据量。

#### 接口命名

- rest风格
- url path使用驼峰、中划线分隔
- queyrString使用驼峰

#### 关键接口使用时间戳

为了避免运营商缓存接口数据，在关键业务的接口url path后加上时间戳，格式ts-{utc绝对秒数}，如：

```
http://mobile.demo.com/mobile/album/ts-13xxxxx?albumId=1&device=android
```

#### 公共参数

- 使用cookie传输，服务端不需要回写cookie
- 参数举例：

```
device      android、chezai_android、iPhone、iPad、wp、pc
deviceId    设备id，android为自定义设备唯一标识。iOS6以前为mac地址，ios7为广告id。
version     客户端版本号  1.0.1 < 1.1.1 < 1.1.12
uid&token, duid&dtoken
channelId,SUP,MAC,IDFA,IMEI,aid
```

- 更多详细参数见：

#### 4.1 cookie信息定义

**以下cookie的参数GET和POST请求都要**（之前有部分字段GET请求没有，如XUM SUP）

| cookie key            | cookie value                                                 | 说明                                                         | 是否必须                     |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------- |
| environmentId&_device | URIEncode(device&deviceId&version)                           | （1）environmentId标注环境：线上=1，开发=2，测试=4； （2）device：设备类型，android、chezai_android、iPhone、iPad、wp、pc；   （3）deviceId: 设备id，android为自定义设备唯一标识；ios6以前为mac地址，ios7为广告id。 | 是                           |
| environmentId&_token  | URIEncode(uid&token)                                         | （1）uid: 用户id；   （2）token: 用户登陆后服务端返回的加密串 | 是                           |
| impl                  | URIEncode(com.demo.*)                                        | 版本区分(android package, iosbundleid)                       | 是                           |
| channel               | URIEncode(a1)                                                | 渠道号（区分各个app市场                                      | 是                           |
| SUP                   | URIEncode(19.8318,22.9122,时间戳),cookie value中有逗号会分隔成两个cookie，所以要URIENCODE | 位置信息（POST请求才需要,拿不到位置信息就不传这个cookie）时间戳是milliseconds单位(相对1970.1.1)，GPS获取经纬度的时间点，逗号分隔（**时间戳为新增字段**） | 否，打开定位服务才有，已弃用 |
| NSUP                  | 详细说明                                                     | 位置信息                                                     | 否，打开定位服务才有         |
| XUM                   | URIEncode(mac)，mac做过混淆，混淆方法在下面                  | android为Mac地址,IOS使用IDFA                                 | 是                           |
| idfa                  | ios idfa                                                     | IOS才需要传                                                  | 是                           |
| c-oper                | URIEncode(C-OPER)                                            | 运营商（**新增字段**），移动、联通等                         | 是                           |
| net-mode              | URIEncode(NET-MODE)                                          | 联网方式（**新增字段**）wifi，3g，4g等                       | 是                           |
| res                   | URIEncode(400,800)                                           | Width，high分辨率（**新增字段**）                            | 是                           |
| XIM                   | imei码                                                       | android独有                                                  | 是                           |
| AID                   | Base64(android ID)                                           | android独有                                                  | 是                           |
| umid                  | android独有                                                  |                                                              | 是                           |
| manufacturer          | 制造商Xiaomi                                                 | android独有                                                  | 是                           |
| XD                    | 设备指纹                                                     | android独有                                                  | 是                           |
| osversion             | 系统版本号                                                   | 27                                                           | 是                           |
| device_model          | 设备型号                                                     | ARE-AL00                                                     | 是                           |
| freeFlowType          | 免流类型                                                     | 0、1                                                         | 否                           |
| xm_grade              |                                                              |                                                              | 否                           |
| oaid                  |                                                              | 75ffe732-89ed-61ac-ffae-efdaffb7b807                         | 否                           |
| x-abtest-bucketIds    | abtest bucketid                                              | 逗号分隔                                                     | 否                           |

其中mac 地址混淆方法

```
    public static String decodeMac(String mac) {
        if (StringUtils.isEmpty(mac)){
            return null;
        }
        if (mac.contains(":")){
            return mac;
        }
        byte[] bytes = Base64.decodeBase64(mac);
        StringBuilder sb = new StringBuilder();
        int i = 0;
        for (byte b : bytes) {
            if (i > 0) {
                sb.append(":");
            }
            sb.append(Integer.toHexString(b & 0xFF));
            i++;
        }
        return sb.toString();
    }
```

#### 4.2 关于Request的user-agent字段

```
统一格式：
ting_版本号_渠道号_appId/系统的值
渠道号：主app使用，普通版本是0，社区版是1；子app不需要
appid：子app的id；主app不需要。
系统值包括：设备类型，系统版本等
例：
windows phone: ting_v1.0.6_c0 (WP 8.0.10328.0,NOKIA RM-822_apac_vietnam_224)
ios: ting_v1.0.6_c0 (CFNetwork, iPhone OS 6.1.3, iPhone4,1)
android: ting_v1.0.6_c0(LT26i,Android 15)
子app：ting_v1.0.0_ id1 (CFNetwork, iPhone OS 6.1.3, iPhone4,1)
```

格式如上保持不变，括号里的字段含义如下表所述：

| 名称            | 类型   | 说明                  | 是否必须 |
| --------------- | ------ | --------------------- | -------- |
| CFNetwork       | string | ios发送请求的标识     | 是       |
| iPhone OS 6.1.3 | string | 操作系统型号          | 是       |
| iPhone4,1       | string | 手机型号，iphone4 1代 | 是       |

### 返回结果

#### 模型定义

客户端展示层理想状态不应有业务逻辑，所有数据由服务端处理好返回，客户端只负责展示。

#### 通用模型

ret错误码，当ret!=0, msg表示错误提示（客户端以toast方式提示），data中返回业务数据model。

```
{
	"ret":0, 
	"msg":"",
	"data": {
	
	}
}
```

view model应避免平铺，不同实体分开定义

```
"user": {
    "uid":1,
},
"album": {
    "albumId":1,
},
"tracks": [
     {
         "trackId":1,
     }
]
```

#### 业务模型定义

- 字段命名使用驼峰
- 统一名称，不同接口描述相同字段时，名保持一致

#### ret错误码定义

详情见：

#### httpStatus和ret的使用

http status仍需遵循http规范：

```
200: 成功
400: 参数错误
404: 资源不存在
401: 服务器未授权
403: 未授权
500: 服务端异常
...
```

当接口返回结果为异常结果时（包括业务异常、服务异常），http status 不能为200，无法对应具体的http status时，统一返回500。

#### 错误提示增强

提示类型支持：toast提醒、弹框提醒、页面提醒。



正常返回结果中加入alert对象：

```
{
    "ret":300, 
    "msg":"",
    "alert": {
    	"type": "confirm",
		"title": "有新版本啦",
        "description": "订阅功能升级，更贴心，使用更方便",
        "url": "升级地址",
        "pics": ["图片地址，可以有多张"],
        "hasCancelButton": true, 
        "confirmButtonText": "去升级",
        "cancelButtonText": "继续使用"
    }, 
    "data": {

    }
}
```

### 版本管理

#### 客户端版本

客户端版本号格式`1.0.0.1`，服务端只取前三位版本号，第四位忽略。

客户端APP识别，使用device、channelId、packageName作为唯一标示，在版本管理系统中录入最新版本。在APP启动时，客户端调用版本检查接口，判断是否需要升级。

#### 接口版本管理

当某个接口（功能）升级，对老版本无法兼容或者希望用户升级到新版本时，可在用户使用该接口（功能）时提醒升级，只要返回特定格式的提示即可，参考错误提示增强中的版本升级项。

### 接口安全

#### 传输安全

- 签名、防重放，参考
- 客户端设备信息，不能明文传输包括mac地址、imei、经纬度等能唯一定位用户的隐私数据。

#### 数据安全

- 接口不能删除、修改字段，可以增加字段

### 测试

- 接口开发完成，除单元测试外，需要做接口测试，使用postman、curl等工具。

### 验收

- 客户端对接完成，服务端开发、产品、测试都需要验收。