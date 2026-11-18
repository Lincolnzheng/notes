# cJSON 接口与使用示例

## 简单介绍

cJSON 是一个 C 语言 JSON 库，用于创建和解析 JSON 数据，常见于 MQTT 消息、HTTP 通信和配置文件。使用时，将同一版本的 `cJSON.c` 和 `cJSON.h` 加入工程。

## 简单创建例子

创建一个包含设备名称和温度的对象，并转换成 JSON 字符串输出。

```c
#include <stdio.h>
#include "cJSON.h"

void JsonExample(void)
{
    cJSON *root = cJSON_CreateObject();
    char *text;

    if (root == NULL) {
        return;
    }

    if (cJSON_AddStringToObject(root, "device", "STM32") == NULL ||
        cJSON_AddNumberToObject(root, "temperature", 25.5) == NULL) {
        cJSON_Delete(root);
        return;
    }

    text = cJSON_PrintUnformatted(root);
    if (text != NULL) {
        printf("%s\n", text);
        cJSON_free(text);    /* 释放生成的字符串。 */
    }

    cJSON_Delete(root);      /* 释放对象及其子节点。 */
}
```

输出：

```json
{"device":"STM32","temperature":25.5}
```

## 常用接口介绍

### 创建和添加

| 接口 | 作用 |
| --- | --- |
| `cJSON_CreateObject()` | 创建空对象 `{}`，失败返回 `NULL` |
| `cJSON_CreateArray()` | 创建空数组 `[]` |
| `cJSON_CreateString(str)` | 创建字符串节点 |
| `cJSON_CreateNumber(num)` | 创建数值节点 |
| `cJSON_CreateBool(value)` | 创建布尔节点，0 为假，非 0 为真 |
| `cJSON_AddStringToObject(obj, key, str)` | 添加字符串字段，成功返回新节点，失败返回 `NULL` |
| `cJSON_AddNumberToObject(obj, key, num)` | 添加数值字段 |
| `cJSON_AddBoolToObject(obj, key, value)` | 添加布尔字段 |
| `cJSON_AddItemToObject(obj, key, item)` | 将已有节点加入对象，成功返回非 0 |
| `cJSON_AddItemToArray(array, item)` | 将已有节点加入数组，成功返回非 0 |

### 解析和读取

| 接口 | 作用 |
| --- | --- |
| `cJSON_Parse(text)` | 解析以 `\0` 结尾的 JSON 字符串，成功返回根节点，失败返回 `NULL` |
| `cJSON_ParseWithLength(buf, len)` | 按指定字节数解析数据 |
| `cJSON_GetObjectItemCaseSensitive(obj, key)` | 根据键名查找字段，区分大小写，未找到返回 `NULL` |
| `cJSON_GetArraySize(array)` | 获取数组元素数量 |
| `cJSON_GetArrayItem(array, index)` | 获取数组元素，下标从 0 开始 |
| `cJSON_IsString(item)` | 判断是否为字符串节点 |
| `cJSON_IsNumber(item)` | 判断是否为数值节点 |
| `cJSON_IsBool(item)` | 判断是否为布尔节点 |
| `cJSON_GetStringValue(item)` | 获取字符串值，类型不符返回 `NULL` |
| `cJSON_GetNumberValue(item)` | 获取 `double` 数值，类型不符返回 NaN |

读取字段时，先查找、判断类型，再取值。节点的 `valuestring` 表示字符串值，`valuedouble` 表示数值，`string` 表示对象成员的键名。

### 修改、输出和释放

| 接口 | 作用 |
| --- | --- |
| `cJSON_SetNumberValue(item, num)` | 修改已有数值节点的值 |
| `cJSON_SetValuestring(item, str)` | 修改已有字符串节点的值，失败返回 `NULL` |
| `cJSON_DeleteItemFromObjectCaseSensitive(obj, key)` | 删除指定字段并释放内存 |
| `cJSON_Print(root)` | 生成带缩进、换行的 JSON 字符串，失败返回 `NULL` |
| `cJSON_PrintUnformatted(root)` | 生成紧凑 JSON 字符串，适合通信发送 |
| `cJSON_free(text)` | 释放生成的 JSON 字符串 |
| `cJSON_Delete(root)` | 释放节点及其拥有的子节点 |

记住：**输出字符串和节点树需要分别释放。** 普通节点成功加入父节点后，由父节点统一管理；查找得到的子节点和内部字符串不要单独释放。

参考：[cJSON 官方接口说明](https://github.com/DaveGamble/cJSON/tree/v1.7.19)。
