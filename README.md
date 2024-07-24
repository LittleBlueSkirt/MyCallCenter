# MyCallCenter
Questions when i program

使用多表结构确实会让查询变得复杂一些，但 Django ORM 提供了强大的工具来简化这些操作。你可以利用 Django 的外键和预取机制来高效地进行关联查询。下面是如何使用多表结构并进行查询的详细说明。

### 模型定义

```python
from django.db import models

class CustomerTable(models.Model):
    name = models.CharField(max_length=100)

class StringField(models.Model):
    customer_table = models.ForeignKey(CustomerTable, related_name='string_fields', on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    is_multiline = models.BooleanField(default=False)

class ChoiceField(models.Model):
    customer_table = models.ForeignKey(CustomerTable, related_name='choice_fields', on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    choices = models.TextField()  # 用逗号分隔的选项
    is_multiple = models.BooleanField(default=False)
```

### 序列化器定义

```python
from rest_framework import serializers
from .models import CustomerTable, StringField, ChoiceField

class StringFieldSerializer(serializers.ModelSerializer):
    class Meta:
        model = StringField
        fields = '__all__'

class ChoiceFieldSerializer(serializers.ModelSerializer):
    class Meta:
        model = ChoiceField
        fields = '__all__'

class CustomerTableSerializer(serializers.ModelSerializer):
    string_fields = StringFieldSerializer(many=True)
    choice_fields = ChoiceFieldSerializer(many=True)

    class Meta:
        model = CustomerTable
        fields = '__all__'

    def create(self, validated_data):
        string_fields_data = validated_data.pop('string_fields')
        choice_fields_data = validated_data.pop('choice_fields')
        customer_table = CustomerTable.objects.create(**validated_data)
        for string_field_data in string_fields_data:
            StringField.objects.create(customer_table=customer_table, **string_field_data)
        for choice_field_data in choice_fields_data:
            ChoiceField.objects.create(customer_table=customer_table, **choice_field_data)
        return customer_table
```

### 视图定义

```python
from rest_framework import viewsets
from .models import CustomerTable
from .serializers import CustomerTableSerializer

class CustomerTableViewSet(viewsets.ModelViewSet):
    queryset = CustomerTable.objects.all()
    serializer_class = CustomerTableSerializer
```

### 路由配置

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import CustomerTableViewSet

router = DefaultRouter()
router.register(r'customer_tables', CustomerTableViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

### 查询

为了高效地查询并返回所有相关的数据，可以使用 `select_related` 和 `prefetch_related`。`select_related` 用于一对一和外键关系，而 `prefetch_related` 用于多对多关系和反向外键关系。

在视图中，可以这样调整查询集：

```python
class CustomerTableViewSet(viewsets.ModelViewSet):
    queryset = CustomerTable.objects.all().prefetch_related('string_fields', 'choice_fields')
    serializer_class = CustomerTableSerializer
```

这将确保在查询 `CustomerTable` 时，一并获取相关的 `StringField` 和 `ChoiceField` 数据，避免多次数据库查询，提高性能。

### 使用示例

#### 创建客户表

请求：
```json
{
    "name": "客户表1",
    "string_fields": [
        {"name": "姓名", "is_multiline": false},
        {"name": "地址", "is_multiline": true}
    ],
    "choice_fields": [
        {"name": "性别", "choices": "男,女", "is_multiple": false},
        {"name": "兴趣", "choices": "音乐,体育,阅读", "is_multiple": true}
    ]
}
```

#### 获取客户表

响应：
```json
[
    {
        "id": 1,
        "name": "客户表1",
        "string_fields": [
            {"id": 1, "customer_table": 1, "name": "姓名", "is_multiline": false},
            {"id": 2, "customer_table": 1, "name": "地址", "is_multiline": true}
        ],
        "choice_fields": [
            {"id": 1, "customer_table": 1, "name": "性别", "choices": "男,女", "is_multiple": false},
            {"id": 2, "customer_table": 1, "name": "兴趣", "choices": "音乐,体育,阅读", "is_multiple": true}
        ]
    }
]
```

### 总结

虽然使用多表结构增加了数据模型的复杂性，但 Django ORM 提供的工具可以简化关联查询，提高查询效率。通过合理使用 `select_related` 和 `prefetch_related`，可以在保持数据模型规范化的同时，确保查询性能。多表结构可以带来更好的数据组织和灵活性，尤其在数据量大且复杂的场景下。
