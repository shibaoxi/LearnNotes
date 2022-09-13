# 在Django中集成 Azure Key values

## 安装包

1. 进入项目虚拟环境

2. 安装Azure AD 库

    ```azurepowershell
    pip install azure-identity
    ```

3. 安装Key Vault 机密库

    ```azurepowershell
    pip install azure-keyvault-secrets
    ```

## 创建代码

在Django 项目目录下找到 settings.py 文件，然后添加如下代码

```python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
secret_client = SecretClient(vault_url='https://bx-test-KV.vault.azure.net/', credential=credential)

secret_properties = secret_client.list_properties_of_secrets()
KV = {}
for secret_property in secret_properties:
    # the list doesn't include values or versions of the secrets
    #print(secret_property.name)
    secret = secret_client.get_secret(secret_property.name)
    KV[secret_property.name] = secret.value
```
