##### 查看RS
    kubectl get rs
##### 查看rs的描述
    kubectl describe rs
##### 删除rs
    kubectl delete rs
使用RS更富有表达力的标签选择器
-- -
### matchExpressions 选择器
    selector:
      matchExpressions:
        - key: app  #此选择器要求该pod包含名为‘app’的标签
          operator: In
          values:
            - demoapp   #标签的值必须时“demoapp”
可以给选择器添加额外的表达式，如，每个表达式都必须包含一个key，一个operator(运算符)，并且可能还有一个values的列表（取决于运算符）。
有四个有效的运算符
1. In:Label的值必须与其中一个指定的Values匹配。
2. NotIn：Label的值与任何指定的values不匹配。
3. Exists: pod 必须包含一个指定名称的标签（值不重要）。使用此运算符时不应指定values字段。
4. DoesNotExist：pod不得包含有指定名称的标签。values属性不得指定。
