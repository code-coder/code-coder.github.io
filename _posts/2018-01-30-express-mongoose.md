---
layout: post
title: '使用Express、Mongoose快速搭建一个项目'
subtitle: ' "Hello World, Hello Blog"'
date: 2018-01-30 21:00:00
author: '张豆豆'
header-img: 'img/post-bg-2015.jpg'
catalog: true
tags:
  - 教程
  - Angular
  - Node.js
  - Express
  - Mongoose
---

> “人生就是瞎折腾 ”

## 前言

闲着无事的时候终于想着要搭一个博客，后来又因为太忙，闲置了一段时间。终于又在一个百无聊赖的时候，准备还是发上第一篇博客吧。

在我准备 2017 年校友会的时候的，我准备做一个 **通讯录** 小系统，一边搭建一边写写记录，顺便也就把搭建过程分享出来。

我是用 Express、Mongoose、Angular4 做的一套完整的系统，虽然后面没用上~ 🤣。

---

##开发准备 ###安装 Node.js

去 Node.js[官网](https://nodejs.org/en/)下载最新的稳定版安装  
###安装 MongoDB

我是用 Mac 👉 [安装教程](https://www.jianshu.com/p/2d0a1ecd0c82)

Windows 👉 [安装教程](https://www.cnblogs.com/cbw-mango/p/7987682.html) ###安装 WebStorm
推荐 👉 [官网下载](https://www.jetbrains.com/webstorm/)，网上[激活](http://blog.csdn.net/voke_/article/details/76418116)方式也有很多

---

##开始搭建项目
首先，安装[Express](http://www.expressjs.com.cn/starter/generator.html)，及 Express 项目生成器  
`$ npm install express -g`  
`$ npm install express-generator -g`

快速生成 Express 项目  
`$ express contact-express`

下载项目依赖  
`$ cd contact-express`  
`$ npm install`

最后，启动这个应用。  
`$ npm start`

不过，这个命令不会监听代码变化实时更新到界面。  
为了开发方便，我们安装 nodemon 来实现 **实时编译**
  
`npm install -g nodemon`

新增 app-script 启动入口，在 package.json 中增加

    "scripts": {
        "start": "node ./bin/www",
        "nodemon start": "nodemon app.js"
    }

开发工具我选择的是 WebStorm。这个工具我就不介绍了，点击左下角 npm 可以打开 npm 窗，双击 **nodemon start** 快速启动。  
使用 WebStorm 打开的目录是这样：

![](/img/in-post/post-express-angular/image.png)

#####安装 Mongoose

`$ npm install mongoose --save`

在根目录下新建一个 config.js，我们把配置放到一个文件中方便管理。

    var config = {
        // debug 为 true 时，用于本地调试
        debug: true,
        cookieSecret: 'contact',
        db: 'mongodb://127.0.0.1/contact',
        host: '127.0.0.1',
        port: 3300,
        mongodb_port: 27017,
        mongodb_db:'contact'
    };
    module.exports = config;

然后新建数据库连接文件 database.js：

    //引入mongoose模块
    var mongoose = require('mongoose');
    var config = require('./config');
    //数据库连接地址  链接到myStudent数据库
    var DB_URL = config.db;
    //数据库连接
    mongoose.connect(DB_URL);

    //连接成功终端显示消息
    mongoose.connection.on('connected', function () {
       console.log('mongoose connection open to ' + DB_URL)
    });
    //连接失败终端显示消息
    mongoose.connection.on('error', function () {
       console.log('mongoose error ')
    });
    //连接断开终端显示消息
    mongoose.connection.on('disconnected', function () {
       console.log('mongoose disconnected')
    });

    //创建一个Schema  每一个schema会一一对应mongo中的collection
    var schema = mongoose.Schema;

    //实例化一个Schema
    var ContactSchema = new schema(
       {
          //设置studentSchema信息的数据格式
          name: {type: String},
          gender: {type: String},  // 男士 或者 女士
          phone: {type: String},  // 电话
          business: {type: String}, // 行业
          address: {type: String}, // 地区
          photo: {type: String}, // 最近照片的地址
          favorite: [], // 收藏联系人
          other: {type: String} // 备注
       },
       //{versionKey: false}是干嘛用？如果不加这个设置，我们通过mongoose第一次创建某个集合时，
       // 它会给这个集合设定一个versionKey属性值，我们不需要，所以不让它显示
       {
          versionKey: false
       }
    );
    //生成一个具体user的model并导出
    //第一个参数是集合名，在数据库中会自动加s
    //把Model名字字母全部变小写和在后面加复数s
    var contact = mongoose.model('Contact', ContactSchema);
    //将Student的model导出
    module.exports = contact;

修改 app.js，增加启动入口：

    var express = require('express');
    var path = require('path');
    var favicon = require('serve-favicon');
    var logger = require('morgan');
    var cookieParser = require('cookie-parser');
    var bodyParser = require('body-parser');
    var config = require('./config');

    var index = require('./routes/index');
    var users = require('./routes/users');

    var app = express();

    app.set('port', process.env.PORT || config.port);
    // view engine setup
    app.set('views', path.join(__dirname, 'views'));
    app.set('view engine', 'ejs');

    app.use(bodyParser.urlencoded({extended:false}));
    app.use(bodyParser.json());

    // 允许跨域访问／／／
    app.all('/api/*', function (req, res, next) {
       res.header('Access-Control-Allow-Origin', '*');
       res.header('Access-Control-Allow-Headers', 'x-Request-with');
       res.header('Access-Control-Allow-Methods', 'PUT,POST,GET,DELETE,OPTIONS');
       res.header('X-Powered-By', '4.15.2');
       res.header('Content-Type', 'application/json;charset=utf-8');
       next()   //执行下一个中间件。
    });

    // uncomment after placing your favicon in /public
    //app.use(favicon(path.join(__dirname, 'public', 'favicon.ico')));
    app.use(logger('dev'));
    app.use(bodyParser.json());
    app.use(bodyParser.urlencoded({extended: false}));
    app.use(cookieParser());
    app.use(express.static(path.join(__dirname, 'public')));

    app.use('/', index);
    app.use('/users', users);

    //服务器监听端口
    app.listen(app.get('port'),function(){
       console.log('Express server lisening on port' + app.get('port'));
    });

    // catch 404 and forward to error handler
    app.use(function (req, res, next) {
       var err = new Error('Not Found');
       err.status = 404;
       next(err);
    });

    // error handler
    app.use(function (err, req, res, next) {
       // set locals, only providing error in development
       res.locals.message = err.message;
       res.locals.error = req.app.get('env') === 'development' ? err : {};

       // render the error page
       res.status(err.status || 500);
       res.render('error');
    });

    module.exports = app;

然后就可以使用 mongoose 啦！

在 routes/index.js 里实现增删改查的 restful 接口：

    var express = require('express');
    var Contact = require('../database').contact;

    var router = express.Router();

    /* GET home page. */
    router.get('/', function (req, res, next) {
        res.render('index');
    });

    router.get('/contactList', function (req, res, next) {
        Contact.find({}).exec(function (error, data) {
            if (error) {
                console.log('数据获取失败' + error);
                res.json({
                    status: '000001',
                    message: '查询失败',
                    //传递返回的数据
                    error: error
                })
            }
            else {
                res.json({
                    status: '000000',
                    message: '查询成功',
                    //传递返回的数据
                    data: data
                })
            }
        })
    });

    // 新增联系人
    router.post('/addContact', function (req, res, next) {
        var contact = new Contact({
            name: req.body.name,
            gender: req.body.gender,  // 男士 或者 女士
            phone: req.body.phone,  // 电话
            business: req.body.business, // 行业
            address: req.body.address, // 地区
            photo: req.body.photo, // 最近照片的地址
            favorite: req.body.favorite || [], // 收藏联系人
            other: req.body.other // 备注
        });
        contact.save(function (error) {
            if (error){
                console.log('数据添加失败:'+error);
                res.json({
                    status:'000001',
                    message:'添加失败',
                    error: error
                })
            }
            else {
                console.log('数据添加成功');
                res.json({
                    status:'000000',
                    message:'添加成功',
                    data:req.body
                })
            }
        });
    });

    // 删除联系人
    router.get('/removeContact',function (req,res) {
        //mongoose根据指定条件进行删除
        student.remove({_id: req.body.id},function(error){
            if (error){
                console.log('数据获取失败'+error);
                res.json({
                    status:'000001',
                    message:'删除不成功'
                })
            }
            else{
                res.json({
                    status:'000000',
                    message:'删除成功'
                })
            }
        })
    });

    router.post('/updateContact',function (req,res) {
        //查询的条件
        var whereStr={_id:req.body.id};
        //更新的内容
        var updateStr={
            $set:{
                name: req.body.name,
                gender: req.body.gender,  // 男士 或者 女士
                phone: req.body.phone,  // 电话
                business: req.body.business, // 行业
                address: req.body.address, // 地区
                photo: req.body.photo, // 最近照片的地址
                favorite: req.body.favorite || [], // 收藏联系人
                other: req.body.other // 备注
            }
        };
        //对数据库进行更新
        Contact.update(whereStr,updateStr,function (error) {
            if (error){
                console.log('数据修改失败:'+error);
                res.json({
                    status:'000001',
                    message:'修改失败',
                    error: error
                })
            }
            else{
                console.log('数据修改成功');
                res.json({
                    status:'000000',
                    message:'修改成功',
                    data:req.body
                })
            }
        })
    });

    // 根据id查找
    router.post('/modify',function (req,res) {
        //mongoose根据条件进行查找
        student.find({_id: req.body.id}).exec(function (error,data) {
            if (error){
                console.log('数据获取失败'+error)
            }
            else{
                console.log(data);
                res.json({
                    status:'000000',
                    message:'查询成功',
                    data:data
                });
            }
        })
    });

    // 根据姓名查找
    router.post('/findByName',function (req,res) {
        student.find({name: req.body.searchName}).exec(function (error,data) {
            if (error){
                console.log('查询失败'+error);
                res.json({
                    status:'000001',
                    message:'查询失败'
                })
            }
            else{
                res.json({
                    status:'000000',
                    message:'查询成功',
                    data:data
                })
            }
        })
    });

    module.exports = router;

在浏览器输入地址：**localhost:3300/contactList** 就可以看到返回的 json 数据啦！

服务端已经搭建完毕啦，后面我再更新客户端的快速搭建吧~

## 后记

> 身为一个有梦想的程序员，怎么能会没有自己的博客呢~
