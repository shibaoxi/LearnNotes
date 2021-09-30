# Django-admin.py 和 Manage.py 区别

## 简介

django-admin.py 是Django的一个用于管理任务的命令行工具，manage.py 是对django-admin.py简单包装，每个Django Project 里面都会包含一个manage.py

## 语法

django-admin.py \<subcommand> [options]
manage.py \<subcommand> [options]

## subcommand

- startproject 创建一个项目

    ```bash
    django-admin.py startproject demoapp
    ```

- startapp 创建一个app

    ```bash
     cd .\demoproject\
     django-admin.py startapp demoapp
    ```

- runserver 运行开发服务器

    ```bash
    #默认以8000端口运行
    .\manage.py runserver
    #指定端口
    .\manage.py runserver 8080
    ```

- shell 进入dango shell
- makemigrations 生成数据库同步脚本
- migrate 同步数据库
- showmigrations 查看生成的同步数据库脚本
- sqlflush 查看生成的清空数据库的脚本
- sqlmigrate 查看数据库同步的sql语句

-**manage.py特有的命令**

- createsuperuser 创建超级管理员
- changepassword 修改密码
- clearsessions 清除session
