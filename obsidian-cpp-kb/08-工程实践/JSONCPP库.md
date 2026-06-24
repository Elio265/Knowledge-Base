---
tags: [c++, JSONCPP, JSON, 序列化]
created: 2026-06-24
status: 需要补充
---

# JSONCPP库

## 一句话理解
JSONCPP是C++中广泛使用的开源JSON解析和生成库，以 `Json::Value` 为核心数据结构，提供JSON数据的解析（反序列化）、操作和生成（序列化）的完整API。

## 核心原理

### JSON数据格式
JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，支持以下数据类型：

| 类型 | 示例 | 说明 |
|------|------|------|
| 字符串 | `"name"` | 双引号包围的Unicode字符序列 |
| 数字 | `30`、`3.14` | 整数或浮点数 |
| 布尔值 | `true`、`false` | 真或假 |
| 数组 | `[1, 2, 3]` | 有序值的集合 |
| 对象 | `{"k": "v"}` | 无序键值对集合 |
| null | `null` | 空值 |

### Json::Value 核心类
`Json::Value` 是JSONCPP中最核心的类，用于表示任何JSON数据类型。它通过重载 `operator[]` 支持像操作字典或数组一样访问JSON成员。

```cpp
Json::Value root;
root["name"] = "张三";
root["age"] = 18;
root["score"].append(80);
root["score"].append(90);
```

**构造函数**：支持从bool、int、int64、uint、double、string、C字符串、数组、对象等多种类型构造。

**类型转换方法**：`asString()`、`asInt()`、`asFloat()`、`asBool()`、`asCString()` 等。

## 底层实现

### 序列化（Json → String）

| 类 | 说明 |
|----|------|
| `Json::Writer` | 抽象基类，定义序列化接口 |
| `Json::FastWriter` | 快速序列化，输出紧凑的无格式JSON |
| `Json::StyledWriter` | 格式化序列化，输出带缩进和换行的易读JSON（低版本） |
| `Json::StreamWriter` | 写入输出流（高版本推荐） |
| `Json::StreamWriterBuilder` | StreamWriter的构建器 |

**推荐用法**（高版本）：
```cpp
Json::StreamWriterBuilder swb;
std::unique_ptr<Json::StreamWriter> sw(swb.newStreamWriter());
std::ostringstream oss;
sw->write(root, &oss);
std::cout << oss.str() << std::endl;
```

### 反序列化（String → Json）

| 类 | 说明 |
|----|------|
| `Json::Reader` | 低版本反序列化类 |
| `Json::CharReader` | 抽象基类，高版本反序列化接口 |
| `Json::CharReaderBuilder` | CharReader的构建器 |

**推荐用法**（高版本）：
```cpp
Json::CharReaderBuilder crb;
std::unique_ptr<Json::CharReader> cr(crb.newCharReader());
std::string err;
cr->parse(jsonStr.c_str(), jsonStr.c_str() + jsonStr.size(), &root, &err);
```

## 常见应用场景

1. **网络通信**：客户端和服务器之间的数据交换格式
2. **配置文件**：程序配置的JSON格式存储和读取
3. **API通信**：RESTful API的请求和响应数据格式
4. **数据持久化**：将程序数据结构序列化为JSON保存
5. **日志系统**：结构化的日志输出格式

## 容易踩坑的地方

1. **API版本差异**：低版本使用 `Json::Reader`/`Json::FastWriter`/`Json::StyledWriter`，高版本推荐使用 `Json::CharReaderBuilder`/`Json::StreamWriterBuilder`
2. **类型安全**：使用 `asInt()`、`asString()` 前需确保Value的存储类型与实际匹配
3. **空值检查**：访问不存在的键或越界索引会返回默认值，需使用 `isMember()` 或 `isNull()` 检查
4. **字符串编码**：JSON字符串必须是合法的UTF-8序列
5. **大文件处理**：解析超大JSON文件时注意内存占用

## 面试高频问题

1. JSONCPP的序列化和反序列化是如何实现的？
2. Json::Value的底层数据结构是什么？（类似Variant的联合体）
3. FastWriter和StyledWriter的区别和适用场景？
4. 高版本API（CharReaderBuilder/StreamWriterBuilder）和低版本API（Reader/FastWriter）的区别？
5. 如何遍历一个嵌套的JSON对象？

## 关联知识
- [[httplib库]]
- [[不定参数解析]]
