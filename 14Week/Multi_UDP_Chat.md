UDP_Chat.cpp을 해봤다면, 해당 프로그램도 쉽게 해결할 수 있다.  
전반적인 수정점이나 흐름은 비슷하기 때문이다.  
사실 리스트를 추가한 것 외에는 크게 변동점은 없는 프로그램이다.  
  
```C++
#include <winsock2.h>
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#pragma comment(lib, "Ws2_32.lib")

#include <iostream>
#include <string>
#include <vector>
#include <cstring>
#include <limits>
#include <cstdlib>
#include <ws2tcpip.h>   // inet_pton, inet_ntop 포함
#include <conio.h>      // _kbhit(), _getch() 포함

// POSIX socket headers 삭제

#define BUF_SIZE 1024
using namespace std;

struct Peer {
    string name;
    string ipStr;
    int port;
    sockaddr_in addr;
};

void printPrompt(const string& targetName) {
    cout << "[송신:" << targetName << "] > ";
    cout.flush();
}

int main() {
    int retval;

    // 윈속 초기화
    WSADATA wsa;
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
        return 1;

    int sockfd;
    sockaddr_in local_addr{};

    int local_port = 0;
    int peerCount = 0;
    vector<Peer> peers;

    // ===============================
    // 1. 초기 설정 입력
    // ===============================
    cout << "[설정] 내(로컬) 포트 번호를 입력하세요: ";
    cin >> local_port;

    cout << "[설정] 통신할 사용자(피어)의 수를 입력하세요: ";
    cin >> peerCount;

    if (local_port <= 0 || local_port > 65535) {
        cerr << "로컬 포트 번호는 1~65535 사이여야 합니다.\n";
        return 1;
    }
    if (peerCount <= 0) {
        cerr << "최소 1명 이상의 피어가 필요합니다.\n";
        return 1;
    }

    // 남아 있는 개행 제거 후 getline 사용
    cin.ignore(1, '\n');

    peers.reserve(peerCount);
    for (int i = 0; i < peerCount; ++i) { // 위에서 몇 명과 통신할 지 정한 수대로 다 설정함
        Peer p{};
        string portStr;

        cout << "\n[피어 설정] #" << (i + 1) << " 이름: ";
        getline(cin, p.name);
        if (p.name.empty()) {
            p.name = "Peer" + to_string(i + 1);
        }

        cout << "[피어 설정] #" << (i + 1) << " IP 주소: ";
        getline(cin, p.ipStr);

        cout << "[피어 설정] #" << (i + 1) << " 포트 번호: ";
        getline(cin, portStr);
        p.port = stoi(portStr);

        if (p.port <= 0 || p.port > 65535) {
            cerr << "포트 번호는 1~65535 사이여야 합니다.\n";
            return 1;
        }

        // sockaddr_in 채우기
        memset(&p.addr, 0, sizeof(p.addr));
        p.addr.sin_family = AF_INET;
        p.addr.sin_port = htons(p.port);
        if (inet_pton(AF_INET, p.ipStr.c_str(), &p.addr.sin_addr) <= 0) {
            cerr << "잘못된 IP 주소: " << p.ipStr << "\n";
            return 1;
        }

        peers.push_back(p);
    }

    // 현재 기본 송신 대상 인덱스 (0번째로 시작)
    int currentPeerIdx = 0;
    string currentTargetName = peers[currentPeerIdx].name;

    // ===============================
    // 2. UDP 소켓 생성 및 바인드
    // ===============================
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == SOCKET_ERROR) {
        printf("socket() 실패");
        return 1;
    }

    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    local_addr.sin_port = htons(local_port);

    if (bind(sockfd, reinterpret_cast<sockaddr*>(&local_addr),
        sizeof(local_addr)) == SOCKET_ERROR) {
        cerr << "bind() 실패";
        closesocket(sockfd);
        WSACleanup();
        return 1;
    }

    cout << "\n=== UDP 멀티-유저 채팅 시작 ===\n";
    cout << "로컬 포트: " << local_port << "\n";
    cout << "등록된 피어 목록:\n";
    for (size_t i = 0; i < peers.size(); ++i) {
        cout << "  " << (i + 1) << ") " << peers[i].name
            << " - " << peers[i].ipStr << ":" << peers[i].port << "\n";
    }
    cout << "\n명령어:\n";
    cout << "  /list        : 피어 목록 보기\n";
    cout << "  /use N       : N번째 피어를 현재 송신 대상으로 선택 (1 기반)\n";
    cout << "  /all 메시지  : 모든 피어에게 브로드캐스트\n";
    cout << "  /quit        : 프로그램 종료\n\n";

    printPrompt(currentTargetName);

    bool running = true;

    // ===============================
    // 3. select 루프
    // ===============================
    while (running) {
        fd_set readfds;
        FD_ZERO(&readfds);
        //FD_SET(STDIN_FILENO, &readfds);
        FD_SET(sockfd, &readfds);

        //int maxfd = (sockfd > STDIN_FILENO) ? sockfd : STDIN_FILENO;

        // Windows에서 select의 첫 번째 인자는 무시되므로 0으로 전달해도 됨
        TIMEVAL tv;
        tv.tv_sec = 0;
        tv.tv_usec = 100000;   // 0.1초 타임아웃

        int ret = select(0, &readfds, nullptr, nullptr, &tv);
        if (ret == SOCKET_ERROR) {
            cerr << "select() 실패, 오류 코드: " << WSAGetLastError() << endl;
            break;
        }

        // ---------------------------
        // 3-1. 키보드 입력 처리
        // ---------------------------
        while (_kbhit()) {
            string line;
            if (!getline(cin, line)) {
                cerr << "\n입력 스트림 종료. 프로그램 종료.\n";
                running = false; // 해당 반복문 뿐만아니라, 모든 반복문 종료
                break;
            }

            if (line.empty()) {
                printPrompt(currentTargetName);
                continue;
            }

            // 명령어 처리
            if (line[0] == '/') {
                if (line == "/quit") {
                    cout << "종료합니다.\n";
                    running = false; // 해당 반복문 뿐만아니라, 모든 반복문 종료
                    break;
                }
                else if (line == "/list") {
                    cout << "\n[피어 목록]\n";
                    for (size_t i = 0; i < peers.size(); ++i) {
                        cout << "  " << (i + 1) << ") " << peers[i].name
                            << " - " << peers[i].ipStr << ":" << peers[i].port;
                        if (static_cast<int>(i) == currentPeerIdx) {
                            cout << "  <-- 현재 선택";
                        }
                        cout << "\n";
                    }
                }
                else if (line.rfind("/use", 0) == 0) {
                    // 형식: /use N
                    string numStr = line.substr(4);
                    // 앞뒤 공백 제거
                    auto start = numStr.find_first_not_of(" \t");
                    if (start != string::npos) {
                        numStr = numStr.substr(start);
                    }
                    if (!numStr.empty()) {
                        int idx = atoi(numStr.c_str());
                        if (idx >= 1 && idx <= static_cast<int>(peers.size())) {
                            currentPeerIdx = idx - 1;
                            currentTargetName = peers[currentPeerIdx].name;
                            cout << "현재 송신 대상: "
                                << currentTargetName << "\n";
                        }
                        else {
                            cout << "잘못된 인덱스입니다. 1 ~ "
                                << peers.size() << " 범위에서 선택하세요.\n";
                        }
                    }
                    else {
                        cout << "사용법: /use N (예: /use 2)\n";
                    }
                }
                else if (line.rfind("/all", 0) == 0) {
                    // 형식: /all 메시지...
                    string msg = line.substr(4);
                    auto start = msg.find_first_not_of(" \t");
                    if (start != string::npos) {
                        msg = msg.substr(start);
                    }
                    else {
                        msg.clear();
                    }

                    if (msg.empty()) {
                        cout << "브로드캐스트할 메시지가 비어 있습니다.\n";
                    }
                    else {
                        for (const auto& p : peers) {
                            int sent = sendto( // ssize_t -> int
                                sockfd,
                                msg.c_str(),
                                msg.size(),
                                0,
                                reinterpret_cast<const sockaddr*>(&p.addr),
                                sizeof(p.addr)
                            );
                            if (sent == SOCKET_ERROR) {
                                cerr << ("sendto() 실패");
                                running = false; // 해당 반복문 뿐만아니라, 모든 반복문 종료
                                break;
                            }
                        }
                        cout << "모든 피어에게 전송되었습니다.\n";
                    }
                }
                else {
                    cout << "알 수 없는 명령어입니다.\n";
                }

                printPrompt(currentTargetName);
            }

            // 일반 메시지: 현재 선택된 피어에게 전송
            if (!peers.empty()) {
                const Peer& target = peers[currentPeerIdx];
                int sent = sendto( // ssize_t -> int
                    sockfd,
                    line.c_str(),
                    line.size(),
                    0,
                    reinterpret_cast<const sockaddr*>(&target.addr),
                    sizeof(target.addr)
                );

                if (sent < 0) {
                    cerr << "sendto() 실패";
                    running = false; // 해당 반복문 뿐만아니라, 모든 반복문 종료
                    break;
                }
            }
            else {
                cout << "등록된 피어가 없습니다. 메시지를 보낼 수 없습니다.\n";
            }

            printPrompt(currentTargetName);
        }

        // ---------------------------
        // 3-2. 수신 처리
        // ---------------------------
        if (ret > 0 && FD_ISSET(sockfd, &readfds)) {
            char recv_buf[BUF_SIZE + 1];
            sockaddr_in from_addr{};
            socklen_t from_len = sizeof(from_addr);

            int received = recvfrom( // ssize_t -> int
                sockfd,
                recv_buf,
                BUF_SIZE,
                0,
                reinterpret_cast<sockaddr*>(&from_addr),
                &from_len
            );

            if (received == SOCKET_ERROR) { // 조건문 < 0 대신 == SOCKET_ERROR 사용
                cerr << "recvfrom() 실패, 오류 코드: " << WSAGetLastError() << endl;
                break;
            }

            recv_buf[received] = '\0';

            char from_ip[INET_ADDRSTRLEN];
            inet_ntop(AF_INET, &from_addr.sin_addr, from_ip, sizeof(from_ip));
            int from_port = ntohs(from_addr.sin_port);

            // 수신한 주소가 등록된 피어인지 확인
            string fromName = "Unknown";
            for (const auto& p : peers) {
                if (p.addr.sin_addr.s_addr == from_addr.sin_addr.s_addr &&
                    p.addr.sin_port == from_addr.sin_port) {
                    fromName = p.name;
                    break;
                }
            }

            cout << "\n[받음 " << fromName << " (" << from_ip << ":" << from_port
                << ")] " << recv_buf << "\n";

            printPrompt(currentTargetName);
        }
    }

    closesocket(sockfd);
    WSACleanup();
    return 0;
}
```
