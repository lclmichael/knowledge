Java Web 修改通用Http headers
项目概况：JDK8,SpringBoot1.5,Tomcat7<br>
背景：<br>
前两天一个小会上，陈龙龙前辈提出一个问题：Http的返回Header的Date是GMT，时间比东八区，也就是国内时间差八个小时。http标准规定这个问题其实应该归于rfc7231设定的标准，只让用GMT/UTC标准时间。https://tools.ietf.org/html/rfc7231#section-7.1.1.1 <br>
坑爹的是原本的标准是要加时区的，但被军方给用错了，所以就把时区当成错字去掉了。https://tools.ietf.org/html/rfc1123#page-55 5.2.14 <br>
延伸：<br>
JavaWeb中修改Http header很容易，只要在响应请求的逻辑中获取HttpServletResponse中setHeader就行。但关于修改Date，Server这类默认的header，在工程内就需要考虑如何实施了。总是在接口setHeader显然不适合维护，这时候需要考虑找到best practice。
在Tomcat层面：

