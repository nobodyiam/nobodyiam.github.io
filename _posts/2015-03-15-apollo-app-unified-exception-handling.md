---
layout:     post
title:      Restful服务统一异常处理机制
date:       2015-03-15 14:00:00 +0800
summary:    本文主要介绍了大众点评的阿波罗后端Restful服务的统一异常处理机制。
categories:
---

##一、简介
阿波罗手机端是所有销售和销售主管（5000多人）使用的销售终端。销售可以使用阿波罗手机随时随地接收通知、查看业绩、录入拜访、维护客户、方案等信息，大大提高销售的工作效率（日活4000多人）。同时，阿波罗手机端还在公司内率先采用混合模式开发app，在提高开发效率的同时也为其他手机团队积累了框架和经验。

目前阿波罗各个服务提供方都提供了Restful服务供手机端调用。由于Restful服务是通过HTTP形式供手机端来消费的，所以我们设计了一套简单的状态码扩展来实现前后端对异常的统一处理。

下文就会对这一机制做一个详细的介绍。

##二、概览
阿波罗的后端服务和手机app交互的接口是通过HTTP接口返回JSON格式来达成的。

为了使接口在提供正常服务的同时兼顾异常处理，我们约定了使用**HTTP状态码**和**JSON对象状态码**来定义服务状态。

###2.1 HTTP状态码
作为基于HTTP的服务，[HTTP自身的状态码](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)是我们首要可以利用的资源。

目前我们主要使用HTTP状态码来标识具体业务以外的状态，如：

* 200
	* 应用服务正常
* 500
	* 应用服务宕机
* 401
	* 身份验证异常

**请求示例：**

{% highlight http%}
GET /customers HTTP/1.1
Host: xxx.com
{% endhighlight %}

**响应示例：**

{% highlight http%}
HTTP/1.1 200 OK
{% endhighlight %}

###2.2 JSON对象状态码
由于后台系统的复杂性，每个业务除了实际的对象数据之外，还需要返回提示信息给用户，如操作失败，数据验证失败等。

为了使后端服务接口统一化，我们设计了以下的数据格式来满足业务场景的需求。需要注意的是，对以下情况，HTTP状态码都是200。

* **请求正常返回**

	通过设置JSON对象的code为`200`来告知请求得到处理并正常返回，同时在JSON对象的msg字段中放入实际返回的业务对象。
	
{% highlight javascript%}
{
	"code": 200,
	"msg": {
		//业务对象数据
	}
}
{% endhighlight %}

* **请求处理失败**

	通过设置JSON对象的code为`500`来告知请求处理失败，调用方可以通过查阅JSON对象的msg字段来获取具体错误信息。
	
	一般应用于服务自身的异常，如：
	
	* 数据库异常
	* 调用第三方服务异常
	
{% highlight javascript%}
{
	"code": 500,
	"msg": "请求处理失败原因"
}
{% endhighlight %}
	
* **请求校验错误**

	通过设置JSON对象的code为`999`来告知请求校验失败，调用方可以通过查阅JSON对象的msg字段来获取具体校验信息。
	
	一般应用于当请求数据不满足服务所规定的要求，如：
	
	* 用户输入不正确
	* 用户对所请求的业务数据没有权限
	
{% highlight javascript%}
{
	"code": 999,
	"msg": "具体校验信息"
}
{% endhighlight %}

##三、具体实现

###3.1 服务端
####3.1.1 HTTP状态码
对于HTTP状态码，我们应用所需要处理的只有401。
其它两个状态码（200，500）都是服务器(nginx和tomcat)默认的行为。

如[手机App的统一认证机制](http://nobodyiam.com/2015/03/07/apollo-app-unified-authentication-mechanism/)所述，
后端服务是通过filter来统一对请求做安全认证的。所以如果发现请求身份验证没通过，只需要在filter中直接设置响应状态为Unauthorized即可。

{% highlight java%}
@Override
public void doFilter(ServletRequest req, ServletResponse resp,
					 FilterChain filterChain) throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) resp;

    String authToken = request.getHeader("Authorization");
    if(!oauthUtil.validate(authToken)) {
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
        return;
    }
    
    filterChain.doFilter(req, resp);
}
{% endhighlight %}

####3.1.2 JSON对象状态码
相对HTTP状态码而言，JSON对象状态码和业务的耦合程度更高，场景也更复杂。
所以我们需要的是一种尽量对实际业务无侵入、代码开发量最小的实现方式。

在综合了各种方案和实现复杂度后，我们选定了通过Spring MVC的[ControllerAdvice](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html)和[ExceptionHandler](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html)来实现。

大致过程是:

1. 我们在应用中通过ControllerAdvice注册一个全局的Exception handler来捕获所有的应用异常
2. 在Exception handler中，对捕获到的异常进行处理，把异常转换成如[2.2节](#2.2-json对象状态码)中所约定的格式
3. 为了能使Exception handler区分出服务错误和校验错误，我们设计了一个`ValidationException`类来存储所有的校验异常
4. 在业务代码中，只需要专注于自身的业务实现，对于自身或者第三方的服务异常，可以选择捕获后处理或者直接抛出
5. 对于校验错误，只需要构建一个`ValidationException`对象，存入校验信息后抛出即可

以下是具体实现。

**ControllerAdvice**：

{% highlight java %}
@ControllerAdvice
public class RestResponseEntityExceptionHandler extends
        ResponseEntityExceptionHandler {
    private Logger logger = LoggerFactory.getLogger(getClass());
    private ObjectMapper mapper;

    @ExceptionHandler(Exception.class)
    protected ResponseEntity<Object> handleExceptions(Exception ex,
                                                      WebRequest request) {

        HttpStatus status = HttpStatus.OK;
        HttpHeaders headers = new HttpHeaders();
        String bodyOfResponse = "";

		ServiceResult result = new ServiceResult();
        if(ex instanceof ValidationException){
            result.setCode(999);
            result.setMsg(ex.getMessage());
        } else {
        	logger.error(ex.getMessage(), ex);

            result.setCode(500);
            result.setMsg(ex.getMessage());
        }
        
        try {
            bodyOfResponse = mapper.writeValueAsString(result);
        } catch (IOException e) {
        }

		headers.add("Content-Type", "application/json;charset=UTF-8");

        return handleExceptionInternal(ex, bodyOfResponse, headers, status,request);
    }

    @Resource(name = "jsonMapper")
    public void setMapper(ObjectMapper mapper) {
        this.mapper = mapper;
    }
}
{% endhighlight %}

**ValidationException**

{% highlight java %}
public class ValidationException extends RuntimeException {
    public ValidationException(String message){
        super(message);
    }
}

{% endhighlight %}

**请求处理失败示例代码：**

{% highlight java %}
OperationResult result = customerService.createCustomer(customer);
if (!result.successful()) {
    throw new RuntimeException("保存客户失败 - " + result.getComment());
}
{% endhighlight %}

**校验错误示例代码：**

{% highlight java %}
String name = customer.getCustomerName();
if (name == null || "".equals(name)) {
	throw new ValidationException("客户名不能为空");
}
{% endhighlight %}

###3.2 客户端
在客户端，为了统一处理服务端的状态码，我们首先对ajax进行了一个封装。

这个封装和普通的ajax方法不同之处只是在于对服务返回成功（HTTP状态码为200）的情况做了细化，区分出了JSON状态码不为200的情况，
然后调用error callback，并传入解析出的状态码和状态信息。对于正常返回的请求，则直接取出实际业务对象返回。

所以，服务端制定的code和msg这套格式对于实际的业务调用是透明的。

**ajax封装示例：**

{% highlight javascript %}
var noop = function() {};

Efte.ajax = function(options) {
  var success = options.success || noop;
  var error = options.error || noop;

  options.success = function(data) {
    if (data.code != null && data.code != 200) { //解析JSON状态码
      error(data.code, data.msg);
      return;
    }

    success(data.msg);//直接返回业务对象
  }

  //send actual request
};
{% endhighlight %}

**业务调用示例：**

{% highlight javascript %}
Efte.ajax({
   method: method,
   url: url,
   data: data,
   success: function(data) {
     //handle response data
   },
   error: function(status, message) {
     if (status == 999) {
     	//handle validation error
     	return;
     }
     if (status == 500) {
     	//handle application error
     	return;
     }
   }
 });

{% endhighlight %}

#四、小结

通过这样一套机制，我们解决了阿波罗App调用后端服务的统一异常处理问题，同时还收获了以下好处：

* 异常处理统一实现，维护性和扩展性较好
	* 在后端统一由`ExceptionHandler`实现
	* 在前端统一由自定义的ajax封装实现
* 后端实现对业务代码透明，保证了接口的统一行为
	* 业务接入很轻量，只需要抛出异常即可
