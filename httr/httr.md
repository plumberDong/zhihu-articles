#! https://zhuanlan.zhihu.com/p/493282079
# 在R中调用百度API

百度[AI平台](https://cloud.baidu.com/)提供了很多api端口，涉及文字识别、图片识别和自然语言处理等众多功能。这些工具可以帮助研究者对文字、图片等数据进行有效降维。

尽管百度为如何使用这些端口提供了非常详细的技术文档，但可能由于R语言比较小众，文档中并不涉及R语言的使用说明。

本文将以[情感倾向分析](https://cloud.baidu.com/product/nlp_apply/sentiment_classify)API为案例，展示R中调用此端口的简要流程。在内容安排上，我默认读者已对Python等语言调用这些端口的流程有所了解。

## 情感倾向分析的基本功能

根据百度的[技术文档](https://cloud.baidu.com/doc/NLP/s/zk6z52hds)，情感倾向分析是：


> 对只包含单一主体主观信息的文本，进行自动情感倾向性判断（积极、消极、中性），并给出相应的置信度。为口碑分析、话题监控、舆情分析等应用提供基础技术支持，同时支持用户自行定制模型效果调优。

例如，需要分析的句子是“我爱祖国”，该API返回的信息是：

```
{
    "text":"我爱祖国",
    "items":[
        {
            "sentiment":2,    //表示情感极性分类结果
            "confidence":0.90, //表示分类的置信度
            "positive_prob":0.94, //表示属于积极类别的概率
            "negative_prob":0.06  //表示属于消极类别的概率
        }
    ]
}
```

概而言之，就是对句子的情感倾向进行分类（0:负向，1:中性，2:正向），同时告诉我们该分类的可信度情况。

## 获取access_token

根据情感分析API的[说明](https://cloud.baidu.com/doc/NLP/s/zk6z52hds)，调用此端口需要的URL参数有access_token，也就是一个标记用户身份的令牌。

如何申请该令牌？根据百度提供的[技术文档](https://ai.baidu.com/ai-doc/REFERENCE/Ck3dwjhhu)，需要向`https://aip.baidubce.com/oauth/2.0/token`发送请求，附带个人的`client_id`和`client_secret`信息。这两条信息在通过控制台创建应用时会自动生成，复制过来即可。

如果我们使用的是Python，这两条信息应该放在字典内，R中并没有字典这种数据类型，取而代之的是`list`：

```s
query <- list(
  grant_type='client_credentials', # 百度要求的参数，不要改
  client_id="your_id", # 填写自己的client_id
  client_secret="your_secrete" # 填写自己的client_secret
```

然后，我们用`httr`包向对应url发送GET请求即可，`jsonlite`负责处理返回的json格式内容：

```s
library(httr)
library(jsonlite)
library(tidyverse)

res1 <- GET(url = "https://aip.baidubce.com/oauth/2.0/token", query = query) %>%
  content(as="text", encoding="UTF-8") %>% 
  fromJSON(flatten = TRUE)
```

`res1`是`list`格式，对应的`access_token`：

```s
access_token = res1$access_token
```

## 情感分析API

获取到令牌`access_token`后，就可以调用情感分析API了。根据[技术文档](https://cloud.baidu.com/doc/NLP/s/zk6z52hds)，该端口需要我们向`https://cloud.baidu.com/doc/NLP/s/zk6z52hds`发送`POST`请求，要求为：

![Image](https://pic4.zhimg.com/80/v2-a8e29f8863e4796177fd3e40f5744e9b.png)



**URL参数**：首先需要将上一节获得的`access_token`拼接到`URL`当中。注意，由于该API默认我们的待分析语句是*gbk*格式，而R使用的是*utf-8*，拼接url时要多加一个`charset=UTF-8`，来告诉百度我们使用的文本格式是*utf-8*。

```
url = str_glue("https://aip.baidubce.com/rpc/2.0/nlp/v1/sentiment_classify?access_token={token}&charset=UTF-8", 
                       token = access_token)
```

**header**：以list形式即可。

**body**：以list形式即可。


最后发送：

```s
res2 <- POST(url = url,
            config = list('Content-Type' = "application/json"), # config也就是header
            body = query <- list(
                        text = "我爱我的祖国"
                        ),
            encode = "json") %>% 
  content(as="text", encoding="UTF-8") %>% 
  fromJSON(flatten = T)
```

res2也是一个`list`，主要信息放在了`items`对应的一个数据框中：

```s
List of 3
 $ log_id: num 1.2e+18
 $ text  : chr "我爱我的祖国"
 $ items :'data.frame':	1 obs. of  4 variables:
  ..$ positive_prob: int 1
  ..$ confidence   : int 1
  ..$ negative_prob: int 0
  ..$ sentiment    : int 1
```
