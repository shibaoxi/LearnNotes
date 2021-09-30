# Django 中的MySQL配置

1. 安装pymysql

    ```bash
    pip install pymysql
    ```

2. 在项目目录下面编辑 settings.py 文件，然后修改如下内容

    ```python
    DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'hello_django_db', # DB Name
        'USER': 'root', # DB用户
        'PASSWORD': 'password', # 密码
        'HOST': '192.168.100.171', # DB主机地址
        'PORT': '3306', # 端口
        }
    }
    ```

3. 在项目目录下面编辑 \_\_init__.py 文件

    ```python
    import pymysql

    pymysql.install_as_MySQLdb()
    ```

4. 执行 ``` manage.py makemigrations ```
5. 然后执行 ``` manage.py migrate ```
