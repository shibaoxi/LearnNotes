# 使用JMESPath 查询Azure CLI 输出

作为Azure CLI 输出结果中，要适当将其转换成适合查询的JSON进行处理。CLI输出结果为JSON字典或数组。

## 获取字典中的属性

例如通过如下命令获取Azure VM 中所有属性

```bash
#此命令将输出一个字典
az vm show -g QueryDemo -n TestVM -o json
```

通过如下命令查询存储的配置

```bash
az vm show -g QueryDemo -n TestVM --query storageProfile.osDisk.managedDisk
```

将输出如下结果

<img src="https://raw.githubusercontent.com/shibaoxi/shareimg/master/img/20220316154013.png" width=600 />

## 常用命令和使用方法

1. 查询单个值

    ```bash
    az vm show -g QueryDemo -n TestVM --query osProfile.adminUsername
    ```

    输出结果如下

    ```bash
    "demoadmin"
    ```

    此值看上去像有效的单个值，但请注意，" 字符作为输出的一部分返回。 这表示该对象是一个 JSON 字符串。 务必要注意，当直接将此值作为命令输出分配给环境变量时，shell 可能无法解释引号.

    CLI 为实现此目的而提供的最佳输出选项是 tsv（制表符分隔值）。 具体而言，当检索某个只能是单值（不是字典或列表）的值时，tsv 输出保证不带引号。

    ```bash
    az vm show -g QueryDemo -n TestVM --query osProfile.adminUsername -o tsv
    ```

2. 查询布尔值

    对于要查询类型为布尔类型（true，false）的值时，可以先将其转成string然后进行筛选

    ```bash
    az account list --query "[?to_string(isDefault) == 'true' ]"
    ```

3. 获取多个值

    若要获取多个属性，请将表达式以逗号分隔列表的形式置于方括号 [ ]（多选列表）中。

    ```bash
    az vm show -g QueryDemo -n TestVM --query "[name,osProfile.adminUsername,location]"
    ```

    输出结果如下：

    ```json
    [
        "TestVM",
        "demoadmin",
        "eastasia"
    ]
    ```

4. 重新命名查询到的属性

    若要在查询多个值时获取字典而非数组，请使用 { }（多选哈希）运算符： 多选哈希的格式为 {displayName:JMESPathExpression, ...}。 displayName 将是在输出中显示的字符串，JMESPathExpression 是要求值的 JMESPath 表达式。

    ```bash
    az vm show -g rg-deliprod -n DeliVMAd1 --query "{VMName:name,UserName:osProfile.adminUsername,Location:location}"
    ```

    输出结果为：

    ```bash
    {
        "Location": "eastasia",
        "UserName": "deliadmin",
        "VMName": "DeliVMAd1"
    }
    ```

5. 获取数组中的属性

    数组没有自己的属性，但可以对数组进行索引。使用itemarray[num],获取想要的属性。但是你无法保证 CLI 输出是有序的，因此请避免使用索引操作，除非你对顺序很确定，或者不在意获取什么元素。 若要访问数组中元素的属性，请执行下述两个操作中的一个：平展和筛选。

    平展数组的操作通过 [] JMESPath 运算符来完成。 [] 运算符之后的所有表达式都应用到当前数组中的每个元素。 如果 [] 出现在查询的开头，它会平展 CLI 命令结果。 az vm list 的结果可以通过此功能来检查。

    ```bash
    az vm list --query "[].{Name:name, OS:storageProfile.osDisk.osType, admin:osProfile.adminUsername}"
    ```

    与 --output table 输出格式组合使用时，列名会与多选哈希的 displayKey 值匹配：

    ```bash
    az vm list --query "[].{Name:name, OS:storageProfile.osDisk.osType, admin:osProfile.adminUsername}" --output table
    ```

    输出结果为：

    ```json
    Name     OS       Admin
    -------  -------  ---------
    Test-2   Linux    sttramer
    TestVM   Linux    azureuser
    WinTest  Windows  winadmin
    ```

6. 筛选数组

    另一项用于从数组获取数据的操作是筛选。 筛选操作通过 [?...] JMESPath 运算符来完成。 此运算符采用谓词作为其内容。 谓词是指其求值结果可能为 true 或 false 的任何语句。 所含谓词的求值结果为 true 的表达式包括在输出中。

    JMESPath 提供标准的比较运算符和逻辑运算符。 其中包括 <、<=、>、>=、==、!=。 JMESPath 也支持逻辑 AND (&&)、OR (||) 和 NOT (!)。 表达式可以在括号中进行分组，因此可以形成更复杂的谓词表达式。

    ```bash
    az vm list --query "[?storageProfile.osDisk.osType=='Linux'].{Name:name,  admin:osProfile.adminUsername}" --output table
    ```

    输出结果为：

    ```json
    Name    Admin
    ------  ---------
    Test-2  sttramer
    TestVM  azureuser
    ```

    JMESPath 也有内置函数，可以用来进行筛选。 ? contains(string, substring) 就是这样的一个函数，用于查看字符串是否包含子字符串。 在调用函数之前，会对表达式求值，因此第一个参数可以是完整的 JMESPath 表达式。

    >注意：JMESPath 表达式出来的结果可能是其他格式，这时要使用to_string()转换成字符串。

    ```bash
    az vm list  --query "[? contains(to_string(storageProfile.osDisk.managedDisk.storageAccountType), 'SSD')].{Name:name, Storage:storageProfile.osDisk.managedDisk.storageAccountType}" -o json
    ```

    输出结果：

    ```json
    [
        {
            "Name": "VmServer02",
            "Storage": "StandardSSD_LRS"
        }
    ]
    ```

    > starts_with, ends_with 和contains用法相同

7. 对输出结果进行排序

    JMESPath 函数还有另一用途，即根据查询结果进行运算。 任何返回非布尔值的函数都会改变表达式的结果。 例如，可以通过 sort_by(array, &sort_expression) 按属性值对数据排序。 对于应该在后面作为函数的一部分求值的表达式，JMESPath 使用特殊运算符 &。

    ```bash
     az vm list --query "sort_by([].{Name:name, Size:to_string(storageProfile.osDisk.diskSizeGb)},&Size)" --output table
    ```

    输出结果为：

    ```json
    Name               Size
    -----------------  ------
    VmServer02     127
    VMAd1          127
    VmServer03     127
    VmServer04     127
    VmServer05     127
    VmServer06     127
    suezpoc-vm-extend  127
    fabmedical-pim     30
    template           30
    SQLtestmachineADM  null
    ```
