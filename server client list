// server.cpp : list / get 명령 지원 서버
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <windows.h>
#include <iostream>
#include <string>
#include <fstream>

#pragma comment(lib, "ws2_32.lib")
using namespace std;


string GetFileList() // 파일 목록 얻기 함수
{
    WIN32_FIND_DATAA data;

    HANDLE h = FindFirstFileA("*.*", &data); // 현재 폴더의 모든 파일 검색

    string list = "";

    if (h == INVALID_HANDLE_VALUE) // 검색 실패 시 빈 목록 반환
        return list;

    do {
        string name = data.cFileName;

        if (name.find(".txt") != string::npos || // txt 또는 png 파일만 목록에 추가
            name.find(".png") != string::npos)
        {
            list += name + "|";
        }
    } while (FindNextFileA(h, &data)); // 다음 파일 검색

    FindClose(h); // 핸들 해제
    return list;
}


void SendFile(SOCKET client, string filename) // 파일 전송 함수
{
    ifstream file(filename, ios::binary);

    if (!file.is_open()) // 파일이 존재하지 않을 경우
    {
        int size = 0;
        send(client, (char*)&size, sizeof(size), 0); 
        return;
    }

    // 파일 크기 계산
    file.seekg(0, ios::end);
    int size = file.tellg();
    file.seekg(0, ios::beg);

    // 파일 크기 먼저 전송
    send(client, (char*)&size, sizeof(size), 0);

    // 파일 내용 전송
    char buf[1024];
    while (!file.eof())
    {
        file.read(buf, sizeof(buf));
        int r = file.gcount(); // 실제 읽힌 바이트 수
        send(client, buf, r, 0);
    }

    file.close();
}


int main()
{
    // 현재 서버 실행 경로 출력
    char path[512];
    GetCurrentDirectoryA(512, path);
    cout << "[DEBUG] 실행 경로: " << path << endl;


    WSADATA w;
    WSAStartup(MAKEWORD(2, 2), &w);

    SOCKET server = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(9000);

    bind(server, (sockaddr*)&addr, sizeof(addr));
    listen(server, 1);

    cout << "[서버] 클라이언트 접속 대기..." << endl;

    sockaddr_in caddr = {};
    int clen = sizeof(caddr);
    SOCKET client = accept(server, (sockaddr*)&caddr, &clen);

    cout << "[서버] 클라이언트 접속!" << endl;

    while (true) // 클라이언트 명령 반복 처리
    {
        char buf[1024] = {};
        int len = recv(client, buf, sizeof(buf) - 1, 0); // 명령 수신
        if (len <= 0) break;

        string cmd = buf;
        cout << "[서버] 받은 명령: " << cmd << endl;

        if (cmd == "quit") break; // 종료 명령

        if (cmd == "list") // 파일 목록 전송
        {
            string list = GetFileList();
            cout << "[서버] 전송할 목록: " << list << endl;
            send(client, list.c_str(), list.size(), 0);
        }
        else if (cmd.rfind("get ", 0) == 0) // 파일 요청 처리
        {
            string filename = cmd.substr(4);
            cout << "[서버] 전송할 파일: " << filename << endl;
            SendFile(client, filename);
        }
    }

    closesocket(client);
    closesocket(server);
    WSACleanup();
    return 0;
}


// client.cpp : 명령 입력 + 파일 수신 클라
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <winsock2.h>
#include <iostream>
#include <fstream>
#include <string>

#pragma comment(lib, "ws2_32.lib")
using namespace std;

int main()
{
    WSADATA w;
    WSAStartup(MAKEWORD(2, 2), &w);

    SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9000);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (connect(sock, (sockaddr*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        cout << "[클라이언트] 서버 연결 실패\n";
        return 0;
    }

    cout << "[클라이언트] 서버 연결됨!\n";

    while (true) // 클라이언트 명령 처리 루프
    {
        cout << "\n명령 입력 (list / get 파일명 / quit): "; // 사용자 명령 입력
        string cmd;
        getline(cin, cmd);

        send(sock, cmd.c_str(), cmd.size(), 0); // 서버로 명령 전송

        if (cmd == "quit") break; // 종료 명령이면 루프 종료

        if (cmd == "list")
        {
            char buf[2048] = {};
            int len = recv(sock, buf, sizeof(buf) - 1, 0); // 목록 문자열 수신
            buf[len] = 0;

            cout << "[서버 목록] " << buf << endl;
        }
        else if (cmd.rfind("get ", 0) == 0) // get 명령 처리 (파일 다운로드)
        {
            string filename = cmd.substr(4); // 요청한 파일명만 분리

            int size = 0; // 먼저 파일 크기를 수신
            recv(sock, (char*)&size, sizeof(size), 0);

            if (size <= 0) // 서버 측에서 파일이 없다고 응답한 경우
            {
                cout << "[클라이언트] 파일 없음!" << endl;
                continue;
            }

            cout << "[클라이언트] 파일 크기: " << size << endl;

            // 수신 파일 저장
            ofstream out(("recv_" + filename).c_str(), ios::binary);

            char buf[1024];
            int total = 0;

            while (total < size) // 파일 크기만큼 반복 수신
            {
                int r = recv(sock, buf, sizeof(buf), 0);
                out.write(buf, r);
                total += r;
            }

            out.close();
            cout << "[클라이언트] 다운로드 완료 → recv_" << filename << endl;
        }
    }

    closesocket(sock);
    WSACleanup();
    return 0;
}
