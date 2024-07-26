# MyCallCenter
Questions when i program

[Markdown文件用法](doc/howTo.md)

<details>
    <summary>1.如何使用多表结构并进行查询</summary>
使用多表结构确实会让查询变得复杂一些，但 Django ORM 提供了强大的工具来简化这些操作。你可以利用 Django 的外键和预取机制来高效地进行关联查询。下面是如何使用多表结构并进行查询的详细说明。

## 模型定义

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

## 序列化器定义

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
        customer_table.save()
        for string_field_data in string_fields_data:
            StringField.objects.create(customer_table=customer_table, **string_field_data)
        for choice_field_data in choice_fields_data:
            ChoiceField.objects.create(customer_table=customer_table, **choice_field_data)
        return customer_table
```

## 视图定义

```python
from rest_framework import viewsets
from .models import CustomerTable
from .serializers import CustomerTableSerializer

class CustomerTableViewSet(viewsets.ModelViewSet):
    queryset = CustomerTable.objects.all()
    serializer_class = CustomerTableSerializer
```

## 路由配置

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

## 查询

为了高效地查询并返回所有相关的数据，可以使用 `select_related` 和 `prefetch_related`。`select_related` 用于一对一和外键关系，而 `prefetch_related` 用于多对多关系和反向外键关系。

在视图中，可以这样调整查询集：

```python
class CustomerTableViewSet(viewsets.ModelViewSet):
    queryset = CustomerTable.objects.all().prefetch_related('string_fields', 'choice_fields')
    serializer_class = CustomerTableSerializer
```

这将确保在查询 `CustomerTable` 时，一并获取相关的 `StringField` 和 `ChoiceField` 数据，避免多次数据库查询，提高性能。

## 使用示例

### 创建客户表

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

### 获取客户表

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

## 总结

虽然使用多表结构增加了数据模型的复杂性，但 Django ORM 提供的工具可以简化关联查询，提高查询效率。通过合理使用 `select_related` 和 `prefetch_related`，可以在保持数据模型规范化的同时，确保查询性能。多表结构可以带来更好的数据组织和灵活性，尤其在数据量大且复杂的场景下。
</details>


<details>
    <summary>2.动态生成表</summary>
要实现前端根据客户类型创建不同的数据表，最好的解决方案是前后端协作完成动态表单和动态模型生成。下面我将提供一个详细的步骤指南，涵盖前端表单生成、后端模型创建及数据管理。

### 1. 前端：动态表单生成

前端可以使用Vue.js或React.js来动态生成表单。用户可以通过UI界面添加不同类型的字段（例如字符串、单选、多选等）。以下是一个示例，假设我们使用Vue.js：

```vue
<template>
  <div>
    <h2>Create New Table</h2>
    <form @submit.prevent="createTable">
      <div>
        <label>Table Name:</label>
        <input v-model="tableName" required />
      </div>
      <div v-for="(field, index) in fields" :key="index">
        <input v-model="field.name" placeholder="Field Name" required />
        <select v-model="field.type">
          <option value="char">Single Line Text</option>
          <option value="text">Multi Line Text</option>
          <option value="choice">Choice</option>
        </select>
        <button @click="removeField(index)">Remove</button>
      </div>
      <button type="button" @click="addField">Add Field</button>
      <button type="submit">Create Table</button>
    </form>
  </div>
</template>

<script>
export default {
  data() {
    return {
      tableName: '',
      fields: [
        { name: '', type: 'char' }
      ]
    };
  },
  methods: {
    addField() {
      this.fields.push({ name: '', type: 'char' });
    },
    removeField(index) {
      this.fields.splice(index, 1);
    },
    createTable() {
      // Send table structure to backend
      this.$http.post('/api/create-table/', {
        name: this.tableName,
        fields: this.fields
      }).then(response => {
        console.log(response.data);
      }).catch(error => {
        console.error(error);
      });
    }
  }
};
</script>
```

### 2. 后端：Django 动态模型生成

在后端，接收前端的表单定义并动态创建Django模型。使用Django的 `ContentType` 和 `Model` 类来动态生成模型。

#### 设置Django应用

首先，在Django应用中创建一个视图来处理表单提交：

```python
from django.db import models
from django.apps import apps
from django.contrib.contenttypes.models import ContentType
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json

@csrf_exempt
def create_table(request):
    if request.method == 'POST':
        data = json.loads(request.body)
        table_name = data['name']
        fields = data['fields']

        # Create the dynamic model
        model = create_dynamic_model(table_name, fields)

        return JsonResponse({'status': 'success', 'model': model.__name__})

def create_dynamic_model(name, fields):
    attrs = {
        '__module__': 'your_app.models',
    }
    for field in fields:
        field_name = field['name']
        field_type = field['type']
        if field_type == 'char':
            attrs[field_name] = models.CharField(max_length=255)
        elif field_type == 'text':
            attrs[field_name] = models.TextField()
        elif field_type == 'choice':
            attrs[field_name] = models.CharField(max_length=255)  # Simplified for demo

    # Create the model
    model = type(name, (models.Model,), attrs)

    # Register the model with Django's app registry
    app_label = 'your_app'
    model._meta.app_label = app_label
    apps.register_model(app_label, model)

    # Create the table in the database
    with connection.schema_editor() as schema_editor:
        schema_editor.create_model(model)

    return model
```

在 `your_app/models.py` 中添加：

```python
from django.db import models

class DynamicTable(models.Model):
    name = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
```

### 3. 数据操作接口

创建API来管理动态生成的模型的数据。

```python
from rest_framework import viewsets
from rest_framework.response import Response
from rest_framework.decorators import action
from django.apps import apps
from .models import DynamicTable

class DynamicTableViewSet(viewsets.ViewSet):

    @action(detail=False, methods=['post'])
    def add_data(self, request):
        model_name = request.data.get('model_name')
        data = request.data.get('data')

        Model = apps.get_model('your_app', model_name)
        instance = Model.objects.create(**data)

        return Response({'status': 'success', 'data_id': instance.id})

    @action(detail=False, methods=['get'])
    def list_data(self, request):
        model_name = request.query_params.get('model_name')
        Model = apps.get_model('your_app', model_name)
        data = Model.objects.all().values()

        return Response({'status': 'success', 'data': list(data)})
```

### 4. 配置路由

在 `your_app/urls.py` 中配置路由：

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import create_table, DynamicTableViewSet

router = DefaultRouter()
router.register(r'dynamic-tables', DynamicTableViewSet, basename='dynamic-table')

urlpatterns = [
    path('create-table/', create_table, name='create_table'),
    path('', include(router.urls)),
]
```

### 总结

这个解决方案展示了如何通过前端生成动态表单，并在后端根据这些表单定义动态生成Django模型。这样，你可以根据客户的需求创建不同类型的数据表，并可以在这些动态生成的表中进行数据操作。你可以根据具体需求进一步扩展和完善功能。
</details>
