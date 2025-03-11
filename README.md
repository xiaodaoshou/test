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
3.11
#include <iostream>
#include <string_view>
#include <unordered_map>

class HttpRequest {
public:
    std::string_view method;
    std::string_view url;
    std::string_view version;
    std::unordered_map<std::string, std::string> headers;
    std::string body;  // 仍然用 std::string 存储正文

    bool parse(std::string_view raw_request) {
        size_t pos = 0;

        // 解析请求行
        size_t line_end = raw_request.find("\r\n", pos);
        if (line_end == std::string_view::npos || !parseRequestLine(raw_request.substr(pos, line_end - pos))) {
            return false;
        }
        pos = line_end + 2; // 跳过 "\r\n"

        // 解析头部字段
        while ((line_end = raw_request.find("\r\n", pos)) != std::string_view::npos) {
            std::string_view line = raw_request.substr(pos, line_end - pos);
            pos = line_end + 2; // 跳过 "\r\n"

            if (line.empty()) {  // 头部解析结束
                break;
            }
            if (!parseHeader(line)) {
                return false;
            }
        }

        // 解析请求体（如果存在）
        auto it = headers.find("Content-Length");
        if (it != headers.end()) {
            int content_length = std::stoi(it->second);
            if (pos + content_length <= raw_request.size()) {
                body = std::string(raw_request.substr(pos, content_length));  // 只在必要时拷贝
            } else {
                return false;
            }
        }

        return true;
    }

private:
    bool parseRequestLine(std::string_view line) {
        size_t pos1 = line.find(' ');
        if (pos1 == std::string_view::npos) return false;

        size_t pos2 = line.find(' ', pos1 + 1);
        if (pos2 == std::string_view::npos) return false;

        method = line.substr(0, pos1);
        url = line.substr(pos1 + 1, pos2 - pos1 - 1);
        version = line.substr(pos2 + 1);
        return true;
    }

    bool parseHeader(std::string_view line) {
        size_t pos = line.find(": ");
        if (pos == std::string_view::npos) return false;

        std::string key(line.substr(0, pos));       // 需要拷贝存储
        std::string value(line.substr(pos + 2));    // 需要拷贝存储
        headers[std::move(key)] = std::move(value);
        return true;
    }
};

int main() {
    std::string http_request =
        "GET /index.html HTTP/1.1\r\n"
        "Host: www.example.com\r\n"
        "User-Agent: CustomClient/1.0\r\n"
        "Accept: */*\r\n"
        "Content-Length: 13\r\n"
        "\r\n"
        "Hello, world!";

    HttpRequest request;
    if (request.parse(http_request)) {
        std::cout << "Method: " << request.method << "\n";
        std::cout << "URL: " << request.url << "\n";
        std::cout << "Version: " << request.version << "\n";
        std::cout << "Headers:\n";
        for (const auto &header : request.headers) {
            std::cout << "  " << header.first << ": " << header.second << "\n";
        }
        std::cout << "Body: " << request.body << "\n";
    } else {
        std::cerr << "Failed to parse HTTP request\n";
    }

    return 0;
}

