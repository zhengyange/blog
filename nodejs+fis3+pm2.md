### 一、开发中代码发布的痛点
  1. 测中bug随时存在，
  2. 随时解决bug，要随时发布到开发、测试环境，供测试人员验证
  3. 告别copy，告别ftp

### 二、nodejs\fis3\pm2安装

#### 1. nodejs安装---客户端，即本机开发环境
  本机开发环境nodejs安装，相信不需要多讲，现在前端开发离不开node了

### 2. nodejs安装---服务端
  此nodejs需要安装在公司的内网服务器上，linux服务器的版本很多，在些只讲解通过源码安装nodejs
 1> 下载nodejs源码，下载地址：[http://nodejs.org/download/](http://nodejs.org/download/)
 
![244848-849c148eb99334f2.png](http://upload-images.jianshu.io/upload_images/3236253-f0594fb3cad831c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2> 登录到linux服务器，如果能root权限用户，安装会省下很多麻烦，在此我使用非root用户讲解安装
```bash
# 假设登录非root帐户dev
[dev@linux ~]$ 
```
3> 使用ftp或其他工具将下载好的nodejs源码上传到服务器dev的home目录下，估计非root账户其他目录你也上传不了
4> 进到home目录，确保nodejs源码已经上传，解决压源码
```bash
# 如果要执行sudo，确保你有root密码，
sudo tar xvf node-v6.11.2.tar.gz

# cd到解决压后的文件夹内
cd node-v6.11.2

# 进行编译安装，编译阶段大概30分钟，耐心等待
sudo ./configure 
sudo make 
sudo make install

# 验证安装成功
node -v
v6.11.2

npm -v
3.10.10
```

### 三、linux服务端receiver服务配置启动
在此我们使用fis3的服务器接收端的node版本，参考资料: [fis3发布到远端服务器](http://fis.baidu.com/fis3/docs/beginning/debug.html#%E5%8F%91%E5%B8%83%E5%88%B0%E8%BF%9C%E7%AB%AF%E6%9C%BA%E5%99%A8)

1> 下载fis3---receiver，如果服务器装有git，
```bash
$ git clone https://github.com/fex-team/receiver.git
$ cd receiver
$ npm install
$ node server.js # default port 8999, use `node server.js <port>` change port
```
如果没有git的话，就要下载通过ftp工具上传到服务器home目录下，
2> 启动receiver->server.js
如果直接使用node启动server也没什么问题，只是终端会被挂起，退出终端服务会停止，在些介绍使用pm2启动server，[pm2参考资料](https://github.com/Unitech/pm2)

##### pm2安装：
如果是root用户则可执行全局安装
```bash
npm install -g pm2

# 切到receiver目录
cd ~/home/receiver

# 启动server.js
pm2 start server.js
```

非root用户安装
```bash
# 切换到receiver目录
cd ~/home/receiver

# 将pm2安装为receiver的依赖
npm install pm2 -S

# 启动server.js
./node_modules/.bin/pm2 start server.js

```

##### 测试receiver是否启动成功，
浏览器输入服务地址: http://139.224.xx.xx:8999/  8999是默认端口，如果页面返回`I'm ready for that, you know.`说明服务启动成功

### 四、客户端fis3配置
1> 安装fis3
```bash
npm install -g fis3
```
2> 配置fis3，在项目根目录，或者要执行fis3 release命令的目录下配置fis-conf.js，默认文件名不要稿错了
```bash
# 'test'是在fis-conf.js配置中当作参数使用
fis3 release test
```
3> 简单fis-conf.js配置
```js
// 配置不需要发布到服务器的文件
fis.set('project.ignore', [
  'node_modules/**',
  'src/**',
  'fis-conf.js',
  'package.json',
  'README.md',
  'debug.log'
]);

// 注意此处的test
fis.media('test')
	.match('**', {
		release: '$0'
	})
	.match('*', {
  deploy: fis.plugin('http-push', {
    receiver: 'http://139.xx.xxxx:8999/receiver',
    //远端目录
    to: '/home/www/'
  })
})
```