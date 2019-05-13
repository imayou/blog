---
title: 改极验验证的servlet为springmvc
date: 2018-05-25 23:31:51
tags:
---
官方文档：[http://www.geetest.com/install/sections/idx-server-sdk.html#java](http://www.geetest.com/install/sections/idx-server-sdk.html#java)
获取代码： `git clone https://github.com/GeeTeam/gt-java-sdk.git`

down下来的`java`代码如下
```bash
gt-java-sdk\src\com\geetest\sdk\java
  - javaGeetestLib.java
  - web\demo
    - GeetestConfig.java
    - StartCaptchaServlet.java
    - VerifyLoginServlet.java
  - mobiledemo
    - GeetestConfig.java
    - StartCaptchaServlet.java
    - VerifyLoginServlet.java
```
只针对`pc`的就修改`demo`里面的`java`代码 (`修改后不用配置web.xml`)
将`GeetestConfig`里的`captcha_id`和`private_key` 替换为你自己的然后只用修改以下2个文件即可

修改`StartCaptchaServlet.java` 如下
```java
import java.util.UUID;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

/**
 * @category 使用Get的方式
 * 返回challenge和capthca_id
 * 此方式以实现前后端完全分离的开发模式
 */
@RestController
public class StartCaptchaServlet {
	@RequestMapping(value = "/geetest/register", method = RequestMethod.GET)
	public String getG(HttpServletRequest request, HttpServletResponse response) {
		GeetestLib gtSdk = new GeetestLib(GeetestConfig.getGeetest_id(), GeetestConfig.getGeetest_key());
		String geetestid= UUID.randomUUID() + ""; // 自定义geetestid
		int gtServerStatus = gtSdk.preProcess(userid);// 进行验证预处理
		request.getSession().setAttribute(gtSdk.gtServerStatusSessionKey, gtServerStatus);// 将服务器状态设置到session中
		request.getSession().setAttribute("geetestid", geetestid);// 将geetestid设置到session中
		return gtSdk.getResponseStr();
	}
}
```

修改`VerifyLoginServlet.java` 如下
```java
import java.io.IOException;

import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import com.ayou.site.base.action.SimpleActionSupport;

/**
 * @category 使用post方式
 * 返回验证结果
 * request表单中必须包含challenge, validate, seccode
 */
@RestController
public class VerifyLoginServlet extends SimpleActionSupport {
	protected Logger logger = LoggerFactory.getLogger(this.getClass());

	@RequestMapping(value = "/geetest/validate", method = RequestMethod.POST)
	public String postG(ModelMap map, HttpServletRequest request, String geetest_challenge, String geetest_validate, String geetest_seccode) throws IOException {
		GeetestLib gtSdk = new GeetestLib(GeetestConfig.getGeetest_id(), GeetestConfig.getGeetest_key());
		String challenge = geetest_challenge;
		String validate = geetest_validate;
		String seccode = geetest_seccode;
		// 从session中获取gt-server状态
		int gt_server_status_code = (Integer) request.getSession().getAttribute(gtSdk.gtServerStatusSessionKey);
		// 从session中获取userid
		String geetestid = (String) request.getSession().getAttribute("geetestid");
		int gtResult = 0;
		if (gt_server_status_code == 1) {// gt-server正常，向gt-server进行二次验证
			gtResult = gtSdk.enhencedValidateRequest(challenge, validate, seccode, geetestid);
			logger.info(gtResult + "");
		} else {// gt-server非正常情况下，进行failback模式验证
			logger.info("failback:use your own server captcha validate");
			gtResult = gtSdk.failbackValidateRequest(challenge, validate, seccode);
			logger.info(gtResult + "");
		}

		if (gtResult == 1) {
			map.put("status", "success");
			map.put("version", gtSdk.getVersionInfo());
			return renderJsonMsg(map, "验证通过！");
		} else {
			map.put("status", "fail");
			map.put("version", gtSdk.getVersionInfo());
			return renderJsonMsg(map, "验证失败！");
		}
	}
}
```


最后再修改`login.jsp`里面的`ajax请求地址`类似如下：
```js
<script>
	var handlerPopup = function(captchaObj) {
		// 成功的回调
		captchaObj.onSuccess(function() {
			var validate = captchaObj.getValidate();
			$.ajax({
				url : "geetest/validate", // 进行二次验证geetest/validate
				type : "post",
				dataType : "json",
				data : {
					username : $('#username1').val(),
					password : $('#password1').val(),
					geetest_challenge : validate.geetest_challenge,
					geetest_validate : validate.geetest_validate,
					geetest_seccode : validate.geetest_seccode
				},
				success : function(data) {
					if (data && (data.status === "success")) {
						$(document.body).html('<h1>登录成功</h1>');
					} else {
						$(document.body).html('<h1>登录失败</h1>');
					}
				}
			});
		});
		$("#popup-submit").click(function() {
			captchaObj.show();
		});
		// 将验证码加到id为captcha的元素里
		captchaObj.appendTo("#popup-captcha");
		// 更多接口参考：http://www.geetest.com/install/sections/idx-client-sdk.html
	};
	// 验证开始需要向网站主后台获取id，challenge，success（是否启用failback）
	$.ajax({
		url : "geetest/register?t=" + (new Date()).getTime(), // 加随机数防止缓存
		type : "get",
		dataType : "json",
		success : function(data) {
			// 使用initGeetest接口
			// 参数1：配置参数
			// 参数2：回调，回调的第一个参数验证码对象，之后可以使用它做appendTo之类的事件
			initGeetest({
				gt : data.gt,
				challenge : data.challenge,
				product : "popup", // 产品形式，包括：float，embed，popup。注意只对PC版验证码有效
				offline : !data.success
			// 表示用户后台检测极验服务器是否宕机，一般不需要关注
			// 更多配置参数请参见：http://www.geetest.com/install/sections/idx-client-sdk.html#config
			}, handlerPopup);
		}
	});
</script>
```