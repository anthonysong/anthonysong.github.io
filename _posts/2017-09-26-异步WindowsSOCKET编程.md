---
layout: default
title: "异步Windows SOCKET编程"
date: 2017-09-26
categories: Windows
tags: Windows SOCKET
description: TCP异步通信服务器和客户端
---

### 0x01 设计原理
本项目利用异步的socket编程，实现了TCP的客户端和服务端，并且能够正常收发数据，实现相互通信
![socket1](http://101.132.99.228/post_img/socket1.png)

### 0x02 主要函数
    
    ioctlsocket(//设置socket为异步模式
    _In_ SOCKET s,//要设置的socket
    _In_ long cmd,//设置命令
    _Inout_ u_long FAR * argp//输入输出数据
    );

    CreateThread(//用来创建一个响应线程来处理TCP请求
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,
    _In_ SIZE_T dwStackSize,
    _In_ LPTHREAD_START_ROUTINE lpStartAddress,
    _In_opt_ __drv_aliasesMem LPVOID lpParameter,
    _In_ DWORD dwCreationFlags,
    _Out_opt_ LPDWORD lpThreadId
    );
    DWORD WINAPI AnswerThread(LPVOID lparam)//具体的线程处理函数

### 0x03 具体实现

* 多连接的TCP客户端

        #include"stdafx.h"
        #include<winsock2.h>
        #pragma comment(lib,"ws2_32.lib")
        #include <stdlib.h>
        #include<iostream>
        using namespace std;
        #include<string>
        #include<sstream>
        #define ServerIP "127.0.0.1"
        #define ServerPort 9901
        #define BUF_SIZE 1024

        int _tmain(int argc, _TCHAR* argv[]) {
            //声明变量
            SOCKET sHost;
            WSADATA wsd;
            SOCKADDR_IN servAddr;
            char buf[BUF_SIZE];
            int retVal;
            //初始化socket环境
            if (WSAStartup(MAKEWORD(2, 2), &wsd) != 0) {
                printf("WSAStartup failed!"); system("pause");
                return 1;
            }
            //创建用于通信的socket
            sHost = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
            if (sHost == INVALID_SOCKET) {
                printf("socket failed!"); system("pause");
                WSACleanup();
                return -1;
            }
            int iMode = 1;
        retVal = ioctlsocket(sHost, FIONBIO, (u_long FAR*)&iMode);
        if (SOCKET_ERROR == retVal) {
            printf("ioctlsocket failed\n");
            WSACleanup();
            return -1;}
        //设置服务器socket地址
        servAddr.sin_family = AF_INET;
        servAddr.sin_port = htons(ServerPort);
        servAddr.sin_addr.S_un.S_addr = inet_addr(ServerIP);
        int sServerAddLen = sizeof(servAddr);
        //连接到服务器
        while (true) {
            retVal = connect(sHost, (LPSOCKADDR)&servAddr, sizeof(servAddr));
            if (retVal == SOCKET_ERROR) {
                int err = WSAGetLastError();
                if (err == WSAEWOULDBLOCK || err == WSAEINVAL) {
                    continue;}
                else if (err == WSAEISCONN) {
                    break;}
                else {
                    printf("connect failed!\n%d", WSAGetLastError()); system("pause");
                    closesocket(sHost);
                    WSACleanup();
                    return -1;}}}
        while (true) {
                printf("Please input a string to send:");
                string str;
                getline(cin, str);
                ZeroMemory(buf, BUF_SIZE);
                strcpy_s(buf, str.c_str());
                while (true){
                    retVal = send(sHost, buf, strlen(buf), 0);
                    if (SOCKET_ERROR == retVal) {
                        int err = WSAGetLastError();
                        if (WSAEWOULDBLOCK == err) {
                            Sleep(500);
                            continue;}
                        else
                        {printf("send failed!\n"); system("pause");
                            closesocket(sHost);
                            WSACleanup();
                            return -1;
        }} break;   }
        while (true) {  ZeroMemory(buf, sizeof(buf));
                    retVal = recv(sHost, buf, sizeof(buf) + 1, 0);
                    if (SOCKET_ERROR == retVal) {
                        int err = WSAGetLastError();
                        if (WSAEWOULDBLOCK == err) {
                            Sleep(100);
                            continue;}
                        else if (WSAETIMEDOUT == err || WSAENETDOWN == err) {
                            printf("recv failed!\n");
                            WSACleanup();
                            return -1;}}break;}
                printf("recv From Server: %s \n", buf);
                if (strcmp(buf, "quit") == 0) {
                    printf("quit!"); system("pause");
                    break;}}
            closesocket(sHost);
            WSACleanup();
            system("pause");
            return 0;}

* 多连接的TCP服务器端

        多连接的TCP服务器端代码
        #include"stdafx.h"
        #include<winsock2.h>
        #pragma comment(lib,"ws2_32.lib")
        #include <stdlib.h>
        #include<iostream>
        using namespace std;
        #define BUF_SIZE 1024
        DWORD WINAPI AnswerThread(LPVOID lparam) { //线程处理
        char buf[BUF_SIZE];
            int retVal;
            SOCKET sClient = (SOCKET)(LPVOID)lparam;
            while (true) {ZeroMemory(buf, BUF_SIZE);
                retVal = recv(sClient, buf, sizeof(buf) + 1, 0);
                if (SOCKET_ERROR == retVal) {
                    int err = WSAGetLastError();
                    if (err == WSAEWOULDBLOCK) {
                        Sleep(100);
                        continue;}
                    else if (err == WSAETIMEDOUT || err == WSAENETDOWN) {
                        printf("recv failed!\n");
                        closesocket(sClient);
                        WSACleanup();
                        return -1;}
        }
        SYSTEMTIME st;
        GetLocalTime(&st);
        char sDateTime[30];
        sprintf(sDateTime, "%4d-%2d-%2d %2d:%2d:%2d", st.wYear, st.wMonth, st.wDay, st.wHour, st.wMinute, st.wSecond);
        printf("%s,Recv From Client[%s:%d]:%s\n", sDateTime, inet_ntoa(addrClient.sin_addr), addrClient.sin_port, buf);
        if (strcmp(buf, "quit") == 0) {
            retVal = send(sClient, "quit", strlen("quit"), 0);
            break;}
        else
        {   char msg[BUF_SIZE];
            sprintf_s(msg, "Message received - %s", buf);
            while (true) {
                retVal = send(sClient, msg, strlen(msg), 0);
                if (SOCKET_ERROR == retVal) {
                    int err = WSAGetLastError();
                    if (WSAEWOULDBLOCK == err) {
                        Sleep(500);
                        continue;}
        else {              printf("send faied!\n");
                            closesocket(sServer);
                            closesocket(sClient);
                            WSACleanup();
                            return -1;}}break;}}system("pause");}
        int _tmain(int argc, _TCHAR* argv[]){
            WSADATA wsd;
            SOCKET sServer;
            SOCKET sClient;
            int retVal;
            char buf[BUF_SIZE]; 
            if (WSAStartup(MAKEWORD(2, 2), &wsd) != 0) {//初始化winsock2.2
                printf("WSAStartup initialize failed!");
                return 1;}
            sServer = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP); //创建用于监听的 Socket
            if (sServer == INVALID_SOCKET) {
                printf("socket error!");
                WSACleanup();
                return -1;}
            int iMode = 1; //设置socket为非阻塞模式
            retVal = ioctlsocket(sServer, FIONBIO, (u_long FAR*)&iMode);
            if (SOCKET_ERROR == retVal) {
                printf("ioctlsocket failed!\n");
                WSACleanup();           return -1;      }
                SOCKADDR_IN addrServ; //指定服务器SOcket地址
            addrServ.sin_family = AF_INET;
            addrServ.sin_port = htons(9901);
            addrServ.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
            retVal = bind(sServer, (const struct sockaddr*)&addrServ, sizeof(SOCKADDR_IN));
            if (retVal == SOCKET_ERROR) {//绑定Sockets Server到本地地址
                printf("bind error!");
                closesocket(sServer);
                WSACleanup();
                return -1;}
            retVal = listen(sServer, 3); //在Sockets Serverz 上进行监听
            if (retVal == SOCKET_ERROR) {
                printf("listen error!");
                closesocket(sServer);
                WSACleanup();
                exit(-1);}
            printf("TCP server start... \n");//接收来自客户端的请求
            sockaddr_in addrClient;
            int addrClientLen = sizeof(addrClient);
            while (true) {
                sClient = accept(sServer, (struct sockaddr FAR*)&addrClient, &addrClientLen);
                if (sClient == INVALID_SOCKET) {
                    int err = WSAGetLastError();
                    if (WSAEWOULDBLOCK == err) {
                        Sleep(100);
                        continue;}
                    DWORD dwThreadId;
                    CreateThread(NULL, NULL, AnswerThread, (LPVOID)sClient, 0, &dwThreadId);}
                break;}
        system("pause");
            return 0;
        }









