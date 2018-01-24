项目概况：jdk8,springboot1.5,tomcat7
背景：
前两天一个小会上，陈龙龙前辈提出一个问题：http的返回header的Date是GMT，时间比东八区，也就是国内时间差八个小时。
http标准规定这个问题其实应该归于rfc7231设定的标准:https://tools.ietf.org/html/rfc7231#section-7.1.1.1
坑爹的是原本的标准是要加时区的，但被军方给用错了，所以就把时区当成错字去掉了。https://tools.ietf.org/html/rfc1123#page-55 5.2.14章节
延伸：
