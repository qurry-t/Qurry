#include <iostream>
#include <string>
#include <map>
#include <vector>
#include <thread>
#include <mutex>
#include <windows.h>

// 用于存储文件名和文件路径的全局容器
std::map<std::string, std::string> fileMap;

// 用于保护共享资源的互斥锁
std::mutex mtx;

// 递归遍历目录并处理文件
void processDirectory(const std::string& dirPath) {
    WIN32_FIND_DATAA findFileData;
    std::string searchPath = dirPath + "\\*";

    HANDLE hFind = FindFirstFileA(searchPath.c_str(), &findFileData);
    if (hFind == INVALID_HANDLE_VALUE) {
        return;
    }

    do {
        std::string fileName = findFileData.cFileName;
        if (fileName != "." && fileName != "..") {
            std::string filePath = dirPath + "\\" + fileName;

            if (findFileData.dwFileAttributes & FILE_ATTRIBUTE_DIRECTORY) {
                // 如果是子目录，则启动一个新线程递归处理
                std::thread worker(processDirectory, filePath);
                std::cout << "Create thread" << std::endl;
                worker.detach(); // 不等待子线程结束
            }
            else {
                // 如果是文件，将文件名和路径添加到map中
                std::lock_guard<std::mutex> lock(mtx);  // 锁定互斥锁
                fileMap[fileName] = filePath;
            }
        }
    } while (FindNextFileA(hFind, &findFileData) != 0);

    FindClose(hFind);
    std::cout << "End threadn" << std::endl;
}

int main() {
    std::string rootDir = "C:\\Users\\Pcdev_2019\\Desktop\\te";  // 替换成你的目标目录路径

    // 启动一个线程来处理根目录
    std::thread worker(processDirectory, rootDir);
    worker.join(); // 等待根目录线程结束

    // 打印文件名和文件路径
    for (const auto& pair : fileMap) {
        std::cout << "File: " << pair.first << ", Path: " << pair.second << std::endl;
    }

    return 0;
}
