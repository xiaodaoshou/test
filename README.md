http解析

```c++
#include <iostream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <algorithm>

// 定义 HTTP 请求数据结构
struct HttpRequest {
    std::string method;    // 请求方法，如 GET、POST
    std::string uri;       // 请求 URI
    std::string version;   // HTTP 版本
    std::unordered_map<std::string, std::string> headers; // 请求头集合
    std::string body;      // 请求消息体
};

// 辅助函数：去除字符串首尾的空白字符
static inline std::string trim(const std::string &s) {
    auto start = s.find_first_not_of(" \t\r\n");
    auto end = s.find_last_not_of(" \t\r\n");
    return (start == std::string::npos) ? "" : s.substr(start, end - start + 1);
}

// 解析 HTTP 请求
bool parseHttpRequest(const std::string &raw, HttpRequest &request) {
    std::istringstream stream(raw);
    std::string line;

    // 1. 解析请求行
    if (!std::getline(stream, line)) {
        std::cerr << "读取请求行失败" << std::endl;
        return false;
    }
    // 去掉行尾可能存在的 '\r'
    line = trim(line);
    std::istringstream requestLineStream(line);
    if (!(requestLineStream >> request.method >> request.uri >> request.version)) {
        std::cerr << "无效的请求行" << std::endl;
        return false;
    }
    
    // 2. 解析请求头
    while (std::getline(stream, line)) {
        line = trim(line);
        // 遇到空行说明请求头结束
        if (line.empty())
            break;
        // 找到冒号分隔的位置
        size_t pos = line.find(":");
        if (pos == std::string::npos) {
            std::cerr << "无效的请求头: " << line << std::endl;
            continue;
        }
        std::string headerName = trim(line.substr(0, pos));
        std::string headerValue = trim(line.substr(pos + 1));
        request.headers[headerName] = headerValue;
    }
    
    // 3. 解析消息体（如果有的话）
    // 这里简单地读取剩余部分作为消息体，实际应用中可能需要根据 Content-Length 等头信息进行处理
    std::string body;
    while (std::getline(stream, line)) {
        body += line + "\n";
    }
    request.body = body;
    
    return true;

}

// 示例：主函数中调用解析函数
int main() {
    // 一个简单的 HTTP 请求示例
    std::string rawRequest =
        "GET /index.html HTTP/1.1\r\n"
        "Host: www.example.com\r\n"
        "User-Agent: Mozilla/5.0\r\n"
        "Accept: text/html\r\n"
        "\r\n"
        "这里是请求体内容";

    HttpRequest request;
    if (parseHttpRequest(rawRequest, request)) {
        std::cout << "请求方法: " << request.method << std::endl;
        std::cout << "请求 URI: " << request.uri << std::endl;
        std::cout << "HTTP 版本: " << request.version << std::endl;
        std::cout << "请求头:" << std::endl;
        for (const auto &header : request.headers) {
            std::cout << "  " << header.first << ": " << header.second << std::endl;
        }
        std::cout << "消息体:" << std::endl << request.body << std::endl;
    } else {
        std::cerr << "HTTP 请求解析失败" << std::endl;
    }
    
    return 0;

}
```

性能测试

```
#include <iostream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <chrono>
#include <algorithm>

// 定义 HTTP 请求数据结构
struct HttpRequest {
    std::string method;    // 请求方法，如 GET、POST
    std::string uri;       // 请求 URI
    std::string version;   // HTTP 版本
    std::unordered_map<std::string, std::string> headers; // 请求头集合
    std::string body;      // 请求消息体
};

// 辅助函数：去除字符串首尾的空白字符
static inline std::string trim(const std::string &s) {
    auto start = s.find_first_not_of(" \t\r\n");
    auto end = s.find_last_not_of(" \t\r\n");
    return (start == std::string::npos) ? "" : s.substr(start, end - start + 1);
}

// 解析 HTTP 请求
bool parseHttpRequest(const std::string &raw, HttpRequest &request) {
    std::istringstream stream(raw);
    std::string line;

    // 1. 解析请求行
    if (!std::getline(stream, line)) {
        std::cerr << "读取请求行失败" << std::endl;
        return false;
    }
    // 去除行尾可能存在的 '\r'
    line = trim(line);
    std::istringstream requestLineStream(line);
    if (!(requestLineStream >> request.method >> request.uri >> request.version)) {
        std::cerr << "无效的请求行" << std::endl;
        return false;
    }
    
    // 2. 解析请求头
    while (std::getline(stream, line)) {
        line = trim(line);
        // 遇到空行说明请求头结束
        if (line.empty())
            break;
        // 找到冒号分隔的位置
        size_t pos = line.find(":");
        if (pos == std::string::npos) {
            std::cerr << "无效的请求头: " << line << std::endl;
            continue;
        }
        std::string headerName = trim(line.substr(0, pos));
        std::string headerValue = trim(line.substr(pos + 1));
        request.headers[headerName] = headerValue;
    }
    
    // 3. 解析消息体（如果有的话）
    // 这里简单地读取剩余部分作为消息体，实际应用中可能需要根据 Content-Length 等头信息进行处理
    std::string body;
    while (std::getline(stream, line)) {
        body += line + "\n";
    }
    request.body = body;
    
    return true;

}

int main() {
    // 构造一个简单的 HTTP 请求字符串
    std::string rawRequest =
        "GET /index.html HTTP/1.1\r\n"
        "Host: www.example.com\r\n"
        "User-Agent: Mozilla/5.0\r\n"
        "Accept: text/html\r\n"
        "\r\n"
        "这里是请求体内容";

    const int numRequests = 1000000; // 一百万次请求解析
    int successCount = 0;
    
    // 开始计时
    auto startTime = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < numRequests; ++i) {
        HttpRequest request;
        if (parseHttpRequest(rawRequest, request)) {
            ++successCount;
        }
    }
    
    // 结束计时
    auto endTime = std::chrono::high_resolution_clock::now();
    auto durationMs = std::chrono::duration_cast<std::chrono::milliseconds>(endTime - startTime).count();
    
    std::cout << "解析 " << numRequests << " 个请求成功 " << successCount << " 个, 耗时: "
              << durationMs << " 毫秒" << std::endl;
    
    return 0;

}
```

