# Django 中的 Settings.py 常用设置

## 创建一个app

1. 创建一个apps目录

2. 使用如下命令创建一个app

    ```powershell
    django-admin.py startapp demoapp
    ```

3. 把创建的app目录移动到apps目录下面
4. 在 setttings.py 文件中添加如下配置

    ```python
    import os, sys
    
    sys.path.append(os.path.join(BASE_DIR, 'apps'))
    
    INSTALLED_APPS = [
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
        'filemanager', # 注册app
    ]
    
    ```

## 配置数据库

引用key vault 防止写入明文密码

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'StudentDB',
        'USER': KV['sqluser'],
        'PASSWORD': KV['sqlpassword'],
        'HOST': KV['sqlhosturl'],
        'PORT': '3306',
        'OPTIONS': {
            'ssl': {'ca': 'config/DigiCertGlobalRootCA.crt.pem' }
        }
        
    }
}
```

> 如果是用ssl连接需要配置证书

然后在app目录下面找到 __init__.py 文件，添加如下代码

```python
import pymysql

pymysql.install_as_MySQLdb()
```
