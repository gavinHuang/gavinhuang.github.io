---
layout: post
title: "Restful API的权限设计思考"
date: 2015-02-28 10:08:40 +0800
comments: true
categories: 
---
###前言
基于Restful风格的API设计规范已经被广泛接受。鄙公司也不例外，我们的系统基于Micro Service的理念，各个Services（服务）将功能发布为遵循Restful风格的API，每个服务即成为可对外发布的独立单元，服务之间的组合又可成为新的服务，称为BaaS（Back-end as a Service)。

由于各个服务需要单独对外提供服务，并且各个服务之间的业务差别比较大，所以在如何控制资源的权限，确保资源不被意外篡改和非法获取，又不违反Micro-Service的理念（单一业务职责）成为每个服务都要考虑的问题，即如何在开放和安全上做到平衡。

###基本结构

我们总结下来，基本原则当然是遵循Restful规范 （为什么必须遵循Restful规范？如果这是个问题的话，请参考：http://stackoverflow.com/questions/1368014/why-do-we-need-restful-web-services ）。

由于功能API都遵循Restful，所以我们把权限的分为如下几种：

*   读：GET
*   写：PUT
*   新建：POST
*   删除：DELETE
*   获得元信息：HEAD
*   获得对该资源的可用操作：OPTIONS

基于这样的设计，我们把权限信息设计为这样的json结构：
<!-- more -->
目标：resource/_permissions
内容：
<pre><code>
	{
    	"GET": {
        	"user": [
            	"user1",
            	"user2"
        	],
        	"role": [
         	   "role1",
         	   "role2"
        	],
       		"group": [
        	    "group1",
        	    "group2"
        	]
    	},
    	"PUT": {
        	"user": [
            	"user1"
        	],
        	"role": [
            	"role1"
        	],
        	"group": [
            	"group1"
        	]
    	}
	}
</code></pre>

上述的权限设置可以绑定的不同层级的资源上，可以实现对不同租户、不同粒度的资源控制。
然而，这样的设计并不能解决所有问题，下面几个内容是我在工作中遇到的问题以及解决这些问题提的一些思路和实践。

###问题1：面向用户还是面向资源？
面向用户的权限通常围绕用户的三级进行权限分配：企业（租户），组、角色等。这种方案描述的是
<pre><code>”用户拥有哪些资源的权限“</code></pre>

典型的权限信息如下：
<pre><code>	/user/123
	{
	    "user": {
	        "ID": "123",
	        "name": "abc",
	        "permissions": [
	            {
	                "GET": "/product/{id}"
	            },
	            {
	                "PUT": "/product/{id}"
	            }
	        ]
	    }
	}
	</code></pre>
而面向资源的权限则将权限信息直接分配给(Attach to)了一个资源，即通常说的ACL （Access Control List），他描述的是：
<pre><code>”资源可以被哪些用户访问“</code></pre>

两种方案的主要区别在于权限信息分别散落在：

*   各个用户上
*   各个资源上

在基于Micro-Services的架构里，帐号信息通常由专门的服务维护，如果将权限定义为面向用户，意味着用户信息和权限信息有映射关系，而这种关系自然成为帐号服务需要维护的内容。但不同的业务服务有不同的权限类型（GET/PUT/DELETE/POST之外还有许多类型，下文会专门阐述）、资源类型、权限粒度，一个集中的服务很难维护权限信息的全集和变更。

而如果将权限信息与资源绑定，则可以针对不同的资源类型设置不同的权限类型和权限粒度，比如对产品信息，除了基本的增删查改之外，还可以进行“搜索”操作，所以可以在“产品”资源下，增加“Search”的权限类型。

然而，面向资源的权限方案存在粒度过细，维护困难的问题，所以许多系统在此基础上增加了“Policy"的概念，可以简化权限设置。比如Amazon的S3，除了提供ACL，还引入Policy的概念。

Policy不需要特别指定哪些资源对哪些用户开放，只需根据通常的业务模式，划分成如下的几类：

*   Private: Only Owner (usually creator) have permissions to a resource.
*   Public: Other user have permission (read,write) to a resource
*   Public_Read:所有用户可读
*   Public_Write:所有用户可写
*   Owner_Full_Control：资源创建者拥有全部权限

可以说，面向资源的ACL加上面向资源的Policy，为解决Restful风格的API权限问题提供了一个优良的基础。

###问题2：何时需要一个新的权限类型？

上文提到对一个GET权限对应于资源的读取操作，那么如果对”/product“的这样的资源类型（目录？）的GET权限，具体对应业务系统的什么操作呢？

*   1）该资源类型下的所有对象的读权限
*   2）该资源类型的相关属性的读取权限
*   3）列出该资源类型下的所有ID

根据HTTP的标准做法似乎是第三种操作（这也是Amazon S3的做法），但我认为第一种做法却更方便些，可以实现权限的层级结构，简化权限设置。即可以不对”/product“下的所有资源都分别设置GET权限，而只设置”product“的GET权限表示product下各个资源的GET权限。

而对方案1和2种的中的两种操作的权限控制则需要增加新的权限类型：

*   Property
*   List

这里列说我们所有的权限类型：
元操作：

*   GET
*   PUT
*   POST
*   DELETE
*   List
*   Search
*   Run

###问题3：Sub-Resource还是Another Resource
上文提到权限信息最好跟资源绑定，比如，对一个资源 /product/123，有一个专门的信息表述该资源的权限信息，该信息本身也是一种资源，那么如何定义该资源的访问路径呢？一般来说，可以通过以下两种路径来定位

*   /product/123/_permissions 
*   /permissions/product/123

前一种是将权限信息视为产品信息的子资源，而后一种则将权限信息视为一种独立的资源，资源路径里包含产品路径只是作为维护“资源-权限”的映射关系的纽带。
在我们的实践里，通常采用第一种方案，即将包括权限信息在内的其他信息视为产品资源的子资源，这些信息包括：

*   _properties: 属性
*   _configuration：配置
*   _permissions：权限


###问题4：是资源还是动作
问题2提到了几种非标准的HTTP Method的元操作类型：如RUN，Search、List，这些操作并不能直接对应到真实的HTTP method上，即对一个资源的GET操作可以是这样的：
<pre><code>GET /resource/id</code></pre>
而对一类资源（类型）的搜索操作却不能这样：
<pre><code>SEARCH /resource/</code></pre>
因为没有Search这个http method （虽然https://datatracker.ietf.org/doc/rfc5323/ 中说明Search也是一个http method)。

目前我们的做法是将这样的操作（动作）转为一个下划线开头的路径（类似子资源），比如搜索操作转变为这样：
<pre><code>GET /resource/id/_search </code></pre>
或
<pre><code>POST /resource/_search</code></pre>

虽然可以说Search也算http method，但是却并不是所以的软件、设备都支持，所以另外一个方案是采用X-HTTP-Method-Override的方法。
比如下面的请求
<pre><code>
GET /resource/
X-HTTP-Method-Override:Search
</code></pre>
表示对/resource资源的search操作。（在spring中有HiddenHttpMethodFilter，可以根据头信息修改客户端的原始HttpMethod）。


###问题5：选哪条路径
当一个实体（一个资源）可以有多条路径到达时，对外的API应该选用哪条路径？这直接关系到权限控制是否有漏洞，比如权限设置某资源不能被用户访问，但用户却可以通过另外一个URL路径访问到该资源。

比如权限设置规定用户不能访问：
<pre><code>/review/{reviewId}</code></pre>
但有可能用户可以通过如下路径访问同一个资源：
<pre><code>/product/{productId}/review/{reviewId}</code></pre>
类似的问题还有：
<pre><code>
/enterpriseId/user/userId
/enterpriseId/group/groupId/user/userId
/enterpriseId/role/roleId/user/userId
</code></pre>
由于面向资源的权限设置的局限性，同一个实际资源，由不同的url表示时，即认为是不同的资源，也即可以拥有不同的权限，所以要控制不同路径的资源的权限设置的一致性，最简单的办法是在API层面只发布一个路径。而如何在这些路径中选择一个合适的路径问题并没有统一的答案，需要根据实际情况灵活处理，但是有几个原则可以参考：

*   原则一： 是否有从属关系：user和role，group之间，并没有硬性的从属关系，一个用户可以没有group或者role而存在。
*   原则三： 云平台的多租户特性。enterprise的隔离性
*   原则二： 简单，开发友好、测试友好，用户友好

比如上述问题的用户信息，由于用户和组或角色之间并没有从属关系，所以建议以用户信息为根路径为好。

###问题6：当一个操作涉及到多个资源时，如何控制权限？（跨资源操作）
目前为止，权限信息与一个或一类资源绑定，并没有提到涉及到多个资源的操作的权限控制。典型的跨资源操作如：拷贝, 批量删除多个资源。
<pre><code>
delete /resourceType/id1,id2,id3
copy /resourceType/source?to=resourceType/target 
</code></pre>
这个时候Restful的弱点开始显现，因为这类问题是一个业务处理，而不是访问一个资源。但是也有一个折中的办法：

由于copy是一个不常用的、不被广泛支持的的http method，所以可以认为这个操作可以可以分解为：
<pre><code>
对原资源的读取操作 (GET) +目的资源的写入操作 (PUT)
</code></pre>
权限控制器需要分别判断两个资源的相应权限。

当然，如果需要更精细的控制的话，应该有单独的COPY操作，因为除了基本的读和写操作之外，复制的另一个含义是服务器端代替客户端执行读和写操作，比一般的由客户端发起的读写操作更复杂，更占用系统资源。在一般的系统中可以忽略这个问题。

所以从权限设置的角度来说，这类操作应该由多个操作组合而来。

###问题7：如何处理同一个资源的同一类操作的不同方式的权限问题
我最近在一个文件存储服务的设计中碰到的一个问题：
新建（上传）一个文件通过POST方法实现，对应的操作权限也为POST，但是新上传一个文件还有另外一个方式，即新建一个上传任务，并将待上传文件分块上传，最后提交任务完成上传。

这里一个矛盾在于，从理论上来说，一个上传任务也是一个资源，他也有 POST/PUT/DELETE的操作，他的权限应该是独立于对一个资源操作的权限的，但实际上他们却相互关联，只有对一个文件有POST或PUT权限，才能新建一个上传job，并且更新这个job，最后提交这个job；

为了免去维护文件操作权限和文件上传任务权限之间的一致性，目前的方案是：
将上传job的三步视为上传操作的三个子操作，他们跟普通上传操作享有同样的权限、HTTP Method，即：不管是新建job、更新job、提交job，他们在API中对应的http method都为POST（新上传文件）或PUT（更新文件），这样就可以简单判断对文件的新建、更新权限来控制文件上传任务的权限。


###结尾
除了上述的几个问题之外还有许多其他问题，都需要在方便和标准之间做一个平衡。



