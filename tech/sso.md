[上一页](quickstart.md)
[回目录](../README.md)
[下一页](flow.md)

# 概述
快流SSO登录是基于 OAuth 2.0 标准协议实现的，通过双向安全认证、免密登录快流。

## 整体流程
### Step1 申请AppID 和 AppSecret
- 在对接快流引擎之前，首先你需要向快流科技申请 AppID 和 AppSecret，联系公司对接人获取。

### Step2 绑定企业平台
- 如果贵公司与飞书、钉钉、企业微信等有集成、且需要快流与这些平台打通、同步用户/部门等基础信息、则需要
在对应平台申请AppId，AppSecret并发给快流技术进行配置；参考 [飞书对接](飞书对接.md)

### Step3 开发与测试
 先了解如下与快流系统交互流程、整体需要贵公司前端与后端依赖快流提供的client加载快流页面、并完成SSO登录；

#### SSO登录交互流程
- 1、用户访问快流页面时、贵公司的前端应用通过快流的前端组件(qfcore.js)集成加载快流页面 、见[前端集成](#前端集成)；
- 2、调用qfcore.js后，qfcore.js会发起登录快流请求、该请求接口需要贵公司的后端参考[accessToken标准接口](#accessToken标准接口)实现该标准接口、获取到正确的accessToken后、qfcore.js就会完成继续调用快流登录接口、完成
SSO登录，全程用户无感知;
##### 所以集成整体开发工作量就两点：
- 1、贵公司的前端调用qfcore.js加载页面;
- 2、贵公司的后端实现 [accessToken标准接口](#accessToken标准接口);

```mermaid
sequenceDiagram
    actor user as 用户
    participant qflowPage as 快流应用页面
    participant customServer as 企业后端应用
    participant qflowServer as 快流服务

    user ->> qflowPage: 进入快流页面
    qflowPage ->> customServer: 获取 accessToken
    note left of qflowPage: 接口地址配置在前端SDK中<br/>请求中会携带当前页面域下的 cookie 信息
    customServer ->> qflowServer: 获取 accessToken
    note left of customServer: 携带用户信息，参考SDK
    qflowServer ->> customServer: 返回 accessToken
    customServer ->> qflowPage: 返回 accessToken
    qflowPage ->> qflowServer: 通过 accessToken 登录
    qflowServer ->> qflowPage: 返回登录结果，添加 qtoken 到快流对应域下
```

## 前端集成
PC浏览器端采用iframe嵌入的方式加载快流工作台界面，步骤如下：

1、下载[qfcore.js](../source/qfcore.js)文件，放置于工程目录中。qfcore是快流前端基础api库，主要用于自动化登录、获取待办等统计信息。

2、代码中引用qfcore.js
```javascript
// commonjs
const qfcore = require('./qfcore');

// es6
import qfcore from './qfcore';
```

3、qfcore api包括
```javascript
// qfcore版本号，返回版本号string
qfcore.getVersion() 

// qfcore初始化，返回promise对象，then参数为是否已登录boolean
qfcore.init({
  // accessToken通过后端接口获取
  accessToken: '',
  // 终端，比如pc端(pc)、飞书小程序(lark-mini)
  client: 'pc',
  // 运行环境，可以不填，默认为prod，也可填beta，beta主要用于测试
  env: 'beta'
})

// 是否已登录，返回登录boolean
qfcore.isLogged()

// 登录，返回promise对象，then参数为是否已登录boolean
qfcore.login()

// 登出，返回promise对象，then参数为是否登出成功boolean
qfcore.logout()

// 请求任务数量统计，返回promise对象，then参数为统计summary object
qfcore.querySummary()
summary: {
  // 待办
  "allTodo": 0,
  // 已超时
  "overdue": 0,
  // 即将超市
  "soonOverdue": 0,
  // 催办
  "urge": 0
}
```

4、在页面中调用qfcore.init，初始化成功后展示iframe，示例：
```javascript
// react
import React, { useState, useEffect } from 'react';
import request from 'request';
import qfcore from './qfcore';

export default function Page() {
  const [logged, setLogged] = useState(false);

  useEffect(() => {
    // ACCESS_TOKEN_URL从后端获取
    request.post(ACCESS_TOKEN_URL).then((res) => {
      const accessToken = res.data.data;
      qfcore
        .init({
          accessToken: accessToken,
          client: 'pc',
          env: 'beta'
        })
        .then((success) => {
          if (success) {
            setLogged(true);
  
            qfcore.querySummary().then((data) => {
              // summary数据可用于入口处气泡等
              console.log(data);
            });
          }
        });
    });
  }, []);
  return logged && <iframe style={{ width: '100%', height: '100vh' }} src="http://beta.kuaiflow.com/user/embed"></iframe>;
}
```
iframe的src线上生产环境为https://www.kuaiflow.com/user/embed，目前内测阶段，先填写http://beta.kuaiflow.com/user/embed


## accessToken标准接口
样例工程代码:
https://github.com/QFlowTech/kuaiflow-demo

后端工作量：
主要需要参考样例工程中的com.kuaiflow.demo.controller.KuaiFlowController#getAccessToken,
实现该http接口。

### Java-SDK
尽量使用线上最新版本:https://repo1.maven.org/maven2/com/kuaiflow/kuaiflow-client/
```xml
<dependency>
    <groupId>com.kuaiflow</groupId>
    <artifactId>kuaiflow-client</artifactId>
    <version>0.0.1</version>
</dependency>
```

### 获取access_token
```java
@Service
public class KuaiFlowBiz {

	private FlowClient flowClient;

	@Value("${kuai.flow.enterpriseCode}")
	private String enterpriseCode;

	@Value("${kuai.flow.appId}")
	private String appId;

	@Value("${kuai.flow.appSecret}")
	private String appSecret;

	//环境[beta:测试/prod:线上]
	@Value("${kuai.flow.environmentType}")
	private String environmentType;

	@PostConstruct
	public void init() {
		// 实例化一个FlowClient、为了保护密钥安全，建议将密钥设置在环境变量中或者配置文件中。
		// Credential cred = new Credential("enterpriseCode", "appId","appSecret","beta");
		Credential credential = new Credential(enterpriseCode, appId, appSecret,environmentType);
		flowClient = new FlowClient(credential);
	}

	public String getAccessToken() {
		CustomAuthentication customAuthentication = new CustomAuthentication();
		// 企业的用户编码-这里获取当前登录用户的企业userCode、即贵公司的用户唯一Id
		// customAuthentication.setCustomUserCode(UserContext.getUserCode());
		customAuthentication.setCustomUserCode("9910031941");
		// 贵公司使用的三方平台用户编码、如飞书Id
		// customAuthentication.setLinkUserCode(UserContext.getLinkUserCode());
		customAuthentication.setLinkUserCode("c94b1dcd");
		return flowClient.getAccessToken(customAuthentication);
	}

}

```


[上一页](quickstart.md)
[回目录](../README.md)
[下一页](flow.md)