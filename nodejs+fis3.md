	周一公司让基本于[百度鹰眼系统][https://github.com/baidu-openmap-trace/web-demo-v3]开发一个项目，  
	之前对fis3也所了解，那么现在用于开发项目，肯定要在fis3的基础上定制自己的开发环境，调研了解到fis3   
	存在下面几点小问题：  
	1. fis3的内置server无法关闭，需要通过进程杀死，这怎么能容忍呢
	2. fis3的mock无法满足项目需求
	3. fis3的proxy是有问题的，官方也是这么说，所以需要自己配置
	4. 需要定制简洁、方便、多功能的npm命令

### 1. 先配置servere，使用express搭建开发服务器，
```js
var express = require('express');
var path = require('path');
var fs = require('fs');
var fse = require('fs-extra');
var child_process = require('child_process');
var proxyMiddleware = require('http-proxy-middleware')
var config = require('./config');
var proxyTable = config.proxyTable
var app = express();

/*
 * 我们将express的静态资源目录直接定位到我们打包后的目录，
 * 这样有几个好处
 * 1. fis3打包后静态资源地址是/static, /script等绝对地址，  
 *    这样的地址可以和express服务的静态资源地址无缝结合
 * 2. fis3打包后的html文件在打包目录的根目录里，这样express把html文件也当作静态资源请求  
 *    免去配置express路由的问题，新添加html页面也不用重新服务器
 * 3. express模板引擎一类的东西统统不用去配
 */
app.use(express.static(config.assetsPublicPath))

/*
 * 由于fis3的proxy接口转发是有问题的，我们需要自己配置，这也很简单  
 * 我们使用 http-proxy-middleware 这个中间件，配置也很简单，可以访问github进行查看
 */

Object.keys(proxyTable).forEach(function (context) {
  var options = proxyTable[context]
  if (typeof options === 'string') {
    options = { target: options }
  }
  app.use(proxyMiddleware(options.filter || context, options))
})


/*
 * 服务器启动部分，为了使用socket.io我们需要按下面的方式启动服务器
 * 使用socket.io是为了监测到fis3打包后的文件目录有变化，向页面触发重新加载命令
 * 实现代码改动页面自动刷新功能
 * 如果不需要此功能，可能使用express启动服务器
 */
var server = require('http').createServer(app);
var io = require('socket.io')(server);
io.on('connection', function(){});

if(!fs.existsSync(config.assetsRoot)) {
	fs.mkdirSync(config.assetsRoot)
}
/*
 * fis3 release命令会监测文件变化，
 * 特别注意：fis3测控整个项目的文件变化，而fs.watch只监测打包后的文件目录变化
 * 使用fs.watch打包目录变化，触发reload事件
 */
fs.watch(config.assetsRoot, function(eventType, filename) {
	// console.log(eventType, filename)
	io.emit('reload')
})
// server.listen(3000);
var aserver = server.listen(config.port, function() {
	var host = aserver.address().address;
	var port = aserver.address().port;
	console.log('App listening at http://%s:%s', host, port);
	doChildprocess();
})

/*
 * fs3 release部分，执行带监测参数的命令，
 * 这里的命令要使用子进程进行执行了，
 */
function doChildprocess() {
	var cmdCli = 'fis3 release dev -wl -d ./web-demo-v3';
	var cpRelease = child_process.exec(cmdCli);
	cpRelease.stdout.pipe(process.stdout);
}
```

### 2. 简单配置如下
```js
var path = require('path');
var helpers = require('./helpers.js');
var build_env = helpers.getBuildEnv();
var proxyTarget = helpers.getProxyTarget();
module.exports = {
	assetsPublicPath: './web-demo-v3',
	assetsRoot: path.resolve(__dirname, '../web-demo-v3'),
	port: '8081',
	proxyTable: {
		'/web-demo-v3/api': {
	        target: proxyTarget,
	        changeOrigin: true,
	        pathRewrite: {
	          '^/web-demo-v3/api': '/web-demo-v3/api'
	        }
	    },
	}
}
```

### 3. 执行生产环境build操作时部分
```js
var child_process = require('child_process');
var config = require('./config');
var cmdCli = 'fis3 release prod -d ' + config.assetsRoot;
var cpRelease = child_process.exec(cmdCli);
cpRelease.stdout.pipe(process.stdout);

// 这里就很简单了
```

### 4. 为了使页面自动刷新，页面需要配置如下代码
```html
<div id="Manager_content"></div>
<% if(metadata.env!='prod'){ %>
<script src="https://cdn.bootcss.com/socket.io/2.0.4/socket.io.js"></script>
<script type="text/javascript">
    var socket = io();
    socket.on('reload', function() {
        window.location.reload();
    })
</script>
<% } %>
```
	这里就牵到fis3解析ejs语法的问题了，头大了吗？
	fis3的官方文档看了好久，也没找到合适的配置说明，那么只好自己写个插件了，
	[fis3-parser-html-plugin][https://github.com/twolun/fis3-parser-html-plugin]，简单配置使用

fis-conf.js中配置：
```js
.match('/html/(*.html)', {
    release: './$1',
    parser: fis.plugin('html-plugin', {
        env: 'prod'
    })
})
// 
```
其实插件的配置，也就插件后面的参数部分会被传到页面中，供ejs解析
`fis3-parser-html-plugin`插件代码其实很简单，要详细了解就要查看ejs语法了

```js
var ejs = require('ejs');
module.exports = function (content, file, settings) {
	if(!file.isHtmlLike) return content;
	return ejs.render(content, {metadata: settings});
}
// 注意此处的settings，就是fis-conf.js中配置的settings
```

### 5. [fis3-parser-html-plugin][https://github.com/twolun/fis3-parser-html-plugin]
	这个插件是我一冲动写的，太冲动了

### 6. 项目源码地址
[web-demo-v3][https://github.com/twolun/web-demo-v3]