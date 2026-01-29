---
title: Spring中webclient处理SSE-调用扣子的智能体api
date: 2024-12-27 10:49:51
author: chengjt
summary:
img:
cover:
coverImg:
tags:
    - spring
    - SSE
    - webClient
categories:
    - spring
---

# Spring中webclient处理SSE
原始需求是通过spring后台调用扣子的智能体，在小程序上实现和ai对话的功能
大概流程是：
1. 通过小程序建立websocket连接提问，
2. 后台调用扣子智能体的api获取回答，
3. 通过websocket在返回前台 

# 实现方式分析
通过查询文档发现扣子的对话api支持两种获取回答的方式：一是使用流式输出直接在对话接口获取ai的回答；二是等待ai回答完成后，调用对话历史接口获取最终回答。
第二种方式需要调用3个接口：调用对话接口提交问题；调用状态接口轮询判断ai是否回答结束；调用对话历史接口获取最终ai回答的内容。<br/>
最终实现为了简单采用了第一种方式：使用webclient调用对话接口，接收流式的响应，即SSE
# webclient 调用代码

## 0.主要依赖版本
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.5.RELEASE</version>
    <relativePath/>
</parent>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```
## 1.应用程序通过公钥和私钥签署 JWT
这部分参考官方的文档：[OAuth JWT 授权](https://www.coze.cn/docs/developer_guides/oauth_jwt)

```java
public class TestServiceImpl {

      static {
            Security.addProvider(new BouncyCastleProvider());
        }

        public static String getToken() throws Exception {
            // 读取私钥文件
            PrivateKey privateKey = readPrivateKeyFromPem("从官网获取的私钥路径");
            // 创建签名器
            JWTSigner signer = JWTSignerUtil.rs256(privateKey);
            // 创建 JWT
            String token = JWT.create()
                    .setHeader("alg", "RS256") // 固定为RS256
                    .setHeader("typ", "JWT") // 固定为 JWT
                    .setHeader("kid", "OAuth 应用的钥指纹") // OAuth 应用的钥指纹
                    .setPayload("iss", "OAuth 应用的 ID") // OAuth 应用的 ID
                    .setPayload("aud", "api.coze.cn") // 扣子 API 的Endpoint
                    .setPayload("iat", System.currentTimeMillis() / 1000) // JWT开始生效的时间，秒级时间戳
                    .setPayload("exp", (System.currentTimeMillis() / 1000) + 3600) // JWT过期时间，秒级时间戳
                    .setPayload("jti", String.valueOf(System.currentTimeMillis() / 1000)) // 随机字符串，防止重放攻击
                    .setSigner(signer)
                    .sign();

            return token;
        }

        /**
         * 从PEM文件中读取私钥
         * @param pemFilePath
         * @return
         * @throws Exception
         */
        public static PrivateKey readPrivateKeyFromPem(String pemFilePath) throws Exception {
            try (FileReader fileReader = new FileReader(pemFilePath);
                 PEMParser pemParser = new PEMParser(fileReader)) {
                Object object = pemParser.readObject();
                if (object instanceof PrivateKeyInfo) {
                    PrivateKeyInfo privateKeyInfo = (PrivateKeyInfo) object;
                    JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider(BouncyCastleProvider.PROVIDER_NAME);
                    return converter.getPrivateKey(privateKeyInfo);
                } else {
                    throw new IllegalArgumentException("Invalid PEM file format");
                }
            }
        }

}
```
##  2.通过 JWT 获取 Oauth Access Token API，获取访问令牌。
[参考官方文档](https://www.coze.cn/docs/developer_guides/oauth_jwt)

代码实现：
```java

    public static String getOAuthToken() throws BotAuthException {
        String token = null;
        try {
            //这里就是上一步获取的jwt签名的字符串
            token = getToken();
        } catch (Exception e) {
            log.error("getOAuthToken FAIL",e);
            throw new BotAuthException("getOAuthToken FAIL",e);
        }
        log.info("getOAuthToken start");
        String value = "Bearer" +" " + token;
        HttpRequest request = HttpUtil.createPost("https://api.coze.cn/api/permission/oauth2/token")
                .header("Content-Type", "application/json")
                .header("Authorization", value)
                //这俩参数grant_type是固定的，duration_seconds 是token过期时间最大为86399。
                //这里直接写死了
                .body("{\n" +
                        "    \"duration_seconds\": 86399,\n" +
                        "    \"grant_type\": \"urn:ietf:params:oauth:grant-type:jwt-bearer\"\n" +
                        "}");
        log.info("getOAuthToken request AUTHORIZATION {} ",value);
        HttpResponse execute = request.execute();
        log.info("getOAuthToken execute");
        log.info("bot auth result: {}", execute.body());
        //处理errormsg todo
        JSONObject parse = JSON.parseObject(execute.body());
        //返回json实例
        //{
        //    "access_token": "czs_RQOhsc7vmUzK4bNgb7hn4wqOgRBYAO6xvpFHNbnl6RiQJX3cSXSguIhFDzgy****",
        //    "expires_in": 1721135859
        //}
        if (parse.containsKey("access_token")) {
            //获取access_token 
            String oAuthToken = parse.getString("access_token");
            OAuth_Token = token;
            log.info("bot auth success oAuthToken{}",token);
            return oAuthToken;
        }
        log.error("bot auth error {}",execute.body());
        throw new BotAuthException("bot auth error");
    }
```
## 3.webclient调用ai的对话接口

```java

public static void post(String token, String bot_id) throws Exception {
        log.info("askQuestion start");
        //token 是上一步调用获取到的access_token
        //bot_id 是bot的id 官方文档里面有获取的方式
        WebClient webClient = WebClient.builder()
                .baseUrl("https://api.coze.cn")
                .defaultHeader("Content-Type", "application/json")
                .defaultHeader("Authorization",  "Bearer" +" "+ token)
                .build();
        //BotQuestionBo 参考官方的文档里面的bo ，这个就是请求的参数自己构造，
        //特别注意这两个参数
        //    private Boolean stream = true; //是否启用流式输出
        //    private Boolean auto_save_history = true; //是否自动保存对话历史
        BotQuestionBo questionBo = new BotQuestionBo();
        log.info("bot question jsonString {}",questionBo);
        log.info("webClient.post 开始请求大模型");
        Flux<byte[]> stringFlux = webClient.post()
                .uri("/v3/chat")
                .accept(MediaType.TEXT_EVENT_STREAM)
                //转换一下
                .body(BodyInserters.fromValue(questionBo))
                .retrieve()
                //这里有坑，建议用bodyToFlux(byte[].class)，不要用bodyToFlux(String.class),后面有具体解释
                .bodyToFlux(byte[].class); 
//                .bodyToFlux(String.class);
        // 订阅事件
        //里面的三个参数是回调函数，自己实现
        //processEvent 就是处理回答的逻辑
        //processOther 处理error
        //processDown处理调用完毕的逻辑，做一些后置处理
        stringFlux.subscribe(
                reviceData-> processEvent(reviceData),
                error ->processOther(error),
                () -> processDown(null)

        );
    }
```
请求参数的样例
```
{
  "bot_id": "bot_id_dee6a895ac01",
  "user_id": "user_id_4b41cd07e225",
  "stream": true,
  "auto_save_history": true,
  "additional_messages": [
    {
        "type": "text",
        "text": "你好我有一个帽衫，我想问问它好看么，你帮我看看"
    }, {
        "type": "image",
        "file_id": "{{file_id_1}}"
    }, {
        "type": "file",
        "file_id": "{{file_id_2}}"
    },
        {
        "type": "file",
        "file_url": "{{file_url_1}}"
    }]
}

```
## 流式返回的样例
```
# chat - 开始
event: conversation.chat.created
// 在 chat 事件里，data 字段中的 id 为 Chat ID，即会话 ID。
data: {"id": "123", "conversation_id":"123", "bot_id":"222", "created_at":1710348675,compleated_at:null, "last_error": null, "meta_data": {}, "status": "created","usage":null}

# chat - 处理中
event: conversation.chat.in_progress
data: {"id": "123", "conversation_id":"123", "bot_id":"222", "created_at":1710348675, compleated_at: null, "last_error": null,"meta_data": {}, "status": "in_progress","usage":null}

# MESSAGE - 知识库召回
event: conversation.message.completed
data: {"id": "msg_001", "role":"assistant","type":"knowledge","content":"---\nrecall slice 1:xxxxxxx\n","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - function_call
event: conversation.message.completed
data: {"id": "msg_002", "role":"assistant","type":"function_call","content":"{\"name\":\"toutiaosousuo-search\",\"arguments\":{\"cursor\":0,\"input_query\":\"今天的体育新闻\",\"plugin_id\":7281192623887548473,\"api_id\":7288907006982012986,\"plugin_type\":1","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - toolOutput
event: conversation.message.completed
data: {"id": "msg_003", "role":"assistant","type":"tool_output","content":"........","content_type":"card","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - answer is card
event: conversation.message.completed
data: {"id": "msg_004", "role":"assistant","type":"answer","content":"{{card_json}}","content_type":"card","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - answer is normal text
event: conversation.message.delta
data:{"id": "msg_005", "role":"assistant","type":"answer","content":"以下","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

event: conversation.message.delta
data:{"id": "msg_005", "role":"assistant","type":"answer","content":"是","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

...... {{ N 个 delta 消息包}} ......

event: conversation.message.completed
data:{"id": "msg_005", "role":"assistant","type":"answer","content":"{{msg_005 完整的结果。即之前所有 msg_005 delta 内容拼接的结果}}","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}


# MESSAGE - 多 answer 的情况,会继续有 message.delta
event: conversation.message.delta
data:{"id": "msg_006", "role":"assistant","type":"answer","content":"你好你好","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

...... {{ N 个 delta 消息包}} ......

event: conversation.message.completed
data:{"id": "msg_006", "role":"assistant","type":"answer","content":"{{msg_006 完整的结果。即之前所有 msg_006 delta 内容拼接的结果}}","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - Verbose （流式 plugin, 多 answer 结束，Multi-agent 跳转等场景）
event: conversation.message.completed
data:{"id": "msg_007", "role":"assistant","type":"verbose","content":"{\"msg_type\":\"generate_answer_finish\",\"data\":\"\"}","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# MESSAGE - suggestion
event: conversation.message.completed
data: {"id": "msg_008", "role":"assistant","type":"follow_up","content":"朗尼克的报价是否会成功？","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}
event: conversation.message.completed
data: {"id": "msg_009", "role":"assistant","type":"follow_up","content":"中国足球能否出现？","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}
event: conversation.message.completed
data: {"id": "msg_010", "role":"assistant","type":"follow_up","content":"羽毛球种子选手都有谁？","content_type":"text","chat_id": "123", "conversation_id":"123", "bot_id":"222"}

# chat - 完成
event: conversation.chat.completed （chat完成）
data: {"id": "123", "chat_id": "123", "conversation_id":"123", "bot_id":"222", "created_at":1710348675, compleated_at:1710348675, "last_error":null, "meta_data": {}, "status": "compleated", "usage":{"token_count":3397,"output_tokens":1173,"input_tokens":2224}}

event: done （stream流结束）
data: [DONE]

# chat - 失败
event: conversation.chat.failed
data: {
    "code":701231,
    "msg":"error"
}

```

# 遇到的坑
bodyToFlux(String.class)的坑，响应接收不全。<br/>

单次返回的完整数据如下：一共三行 event一行，data一行，最后空行
```
event: conversation.chat.completed /r/n （结尾换行符号）
data: {"id": "123", "chat_id": "123", "conversation_id":"123", "bot_id":"222", "created_at":1710348675, compleated_at:1710348675, "last_error":null, "meta_data": {}, "status": "compleated", "usage":{"token_count":3397,"output_tokens":1173,"input_tokens":2224}} /n（结尾换行符号）
/n （结尾换行符号）最后这里空行
````
使用bodyToFlux(byte[].class)接收到，再转为string收到的是完整的数据，包含event和data。就和上面的返回值一样<br/>
使用bodyToFlux(String.class)接收你只能收到data：后面的json，接收不到event的数据,就像下面这样：
```
{"id": "123", "chat_id": "123", "conversation_id":"123", "bot_id":"222", "created_at":1710348675, compleated_at:1710348675, "last_error":null, "meta_data": {}, "status": "compleated", "usage":{"token_count":3397,"output_tokens":1173,"input_tokens":2224}}
```

简单看了一下，转换方法把event这一行和data：丢弃了。具体的做过滤丢弃的代码没找到，没搞明白具体的逻辑。<br/>

有大佬知道详细的话，麻烦留言告知小弟，不胜感激。


