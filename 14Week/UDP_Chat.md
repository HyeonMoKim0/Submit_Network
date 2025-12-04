기말에서 Multi UDP와 그냥 UDP Chat 사이에 어떤 차이가 있는지 인지할 것  
그리고 select에서 굳이 타임아웃을 넣는 이유는, 없으면 윈도우에서 발생하는 이벤트를 처리할 틈이 없어짐  
그래서 TIMEVAL 이라는 구조체의 변수에 tv_usec = 100000;을 넣음으로써 0.1초의 타임아웃을 통해 이벤트를 처리하도록 조치한 것임  
사실상 타임아웃은 항상 있어야하는 꼴임, 다중으로 돌아가게 하려면.  
내가 수정하기 전 코드
```C++
#include <winsock2.h>
#include <WS2tcpip.h>
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#pragma comment(lib, "Ws2_32.lib")

#include <iostream>
#include <string>
#include <vector>
#include <cstring>
#include <limits>
#include <cstdlib>
#include <ws2tcpip.h>      // [추가] inet_pton, inet_ntop
#include <conio.h> // [추가] _kbhit(), _getch()

// POSIX socket headers
//#include <sys/types.h>
//#include <sys/socket.h>
//#include <netinet/in.h>
//#include <arpa/inet.h>
//#include <unistd.h>

#define BUF_SIZE 1024
using namespace std;

struct Peer {
    std::string name;
    std::string ipStr;
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
    std::vector<Peer> peers;

    // ===============================
    // 1. 초기 설정 입력
    // ===============================
    std::cout << "[설정] 내(로컬) 포트 번호를 입력하세요: ";
    std::cin >> local_port;

    std::cout << "[설정] 통신할 사용자(피어)의 수를 입력하세요: ";
    std::cin >> peerCount;

    if (local_port <= 0 || local_port > 65535) {
        cout << "로컬 포트 번호는 1~65535 사이여야 합니다.\n";
        return 1;
    }
    if (peerCount <= 0) {
        cout << "최소 1명 이상의 피어가 필요합니다.\n";
        return 1;
    }

    // 남아 있는 개행 제거 후 getline 사용
    cin.ignore(1, '\n');

    peers.reserve(peerCount);
    for (int i = 0; i < peerCount; ++i) {
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
            cout << "포트 번호는 1~65535 사이여야 합니다.\n";
            return 1;
        }

        // sockaddr_in 채우기
        memset(&p.addr, 0, sizeof(p.addr));
        p.addr.sin_family = AF_INET;
        p.addr.sin_port = htons(p.port);
        if (inet_pton(AF_INET, p.ipStr.c_str(), &p.addr.sin_addr) <= 0) {
            cout << "잘못된 IP 주소: " << p.ipStr << "\n";
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
    if (sockfd < 0) {
        printf("socket() 실패");
        return 1;
    }

    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    local_addr.sin_port = htons(local_port);

    if (::bind(sockfd, reinterpret_cast<sockaddr*>(&local_addr),
        sizeof(local_addr)) == SOCKET_ERROR) {
        perror("bind() 실패");
        ::closesocket(sockfd);
        WSACleanup();
        return 1;
    }

    std::cout << "\n=== UDP 멀티-유저 채팅 시작 ===\n";
    std::cout << "로컬 포트: " << local_port << "\n";
    std::cout << "등록된 피어 목록:\n";
    for (size_t i = 0; i < peers.size(); ++i) {
        std::cout << "  " << (i + 1) << ") " << peers[i].name
            << " - " << peers[i].ipStr << ":" << peers[i].port << "\n";
    }
    std::cout << "\n명령어:\n";
    std::cout << "  /list        : 피어 목록 보기\n";
    std::cout << "  /use N       : N번째 피어를 현재 송신 대상으로 선택 (1 기반)\n";
    std::cout << "  /all 메시지  : 모든 피어에게 브로드캐스트\n";
    std::cout << "  /quit        : 프로그램 종료\n\n";

    printPrompt(currentTargetName);

    // ===============================
    // 3. select 루프
    // ===============================
    while (true) {
        fd_set readfds;
        FD_ZERO(&readfds);
        //FD_SET(STDIN_FILENO, &readfds);
        FD_SET(sockfd, &readfds);

        //int maxfd = (sockfd > STDIN_FILENO) ? sockfd : STDIN_FILENO;

        // Windows에서 select의 첫 번째 인자는 무시되므로 0으로 전달해도 됨
        TIMEVAL tv;
        tv.tv_sec = 0;
        tv.tv_usec = 100000;   // 0.1초 타임아웃

        int ret = ::select(0, &readfds, nullptr, nullptr, &tv);
        if (ret == SOCKET_ERROR) {
            int err = WSAGetLastError();
            cerr << "select() 실패, 오류 코드: " << err << endl;
            break;
        }

        // ---------------------------
        // 3-1. 키보드 입력 처리
        // ---------------------------
        while (_kbhit()) {
            std::string line;
            if (!std::getline(std::cin, line)) {
                std::cout << "\n입력 스트림 종료. 프로그램 종료.\n";
                break;
            }

            if (line.empty()) {
                printPrompt(currentTargetName);
                continue;
            }

            // 명령어 처리
            if (line[0] == '/') {
                if (line == "/quit") {
                    std::cout << "종료합니다.\n";
                    break;
                }
                else if (line == "/list") {
                    std::cout << "\n[피어 목록]\n";
                    for (size_t i = 0; i < peers.size(); ++i) {
                        std::cout << "  " << (i + 1) << ") " << peers[i].name
                            << " - " << peers[i].ipStr << ":" << peers[i].port;
                        if (static_cast<int>(i) == currentPeerIdx) {
                            std::cout << "  <-- 현재 선택";
                        }
                        std::cout << "\n";
                    }
                }
                else if (line.rfind("/use", 0) == 0) {
                    // 형식: /use N
                    std::string numStr = line.substr(4);
                    // 앞뒤 공백 제거
                    auto start = numStr.find_first_not_of(" \t");
                    if (start != std::string::npos) {
                        numStr = numStr.substr(start);
                    }
                    if (!numStr.empty()) {
                        int idx = std::atoi(numStr.c_str());
                        if (idx >= 1 && idx <= static_cast<int>(peers.size())) {
                            currentPeerIdx = idx - 1;
                            currentTargetName = peers[currentPeerIdx].name;
                            std::cout << "현재 송신 대상: "
                                << currentTargetName << "\n";
                        }
                        else {
                            std::cout << "잘못된 인덱스입니다. 1 ~ "
                                << peers.size() << " 범위에서 선택하세요.\n";
                        }
                    }
                    else {
                        std::cout << "사용법: /use N (예: /use 2)\n";
                    }
                }
                else if (line.rfind("/all", 0) == 0) {
                    // 형식: /all 메시지...
                    std::string msg = line.substr(4);
                    auto start = msg.find_first_not_of(" \t");
                    if (start != std::string::npos) {
                        msg = msg.substr(start);
                    }
                    else {
                        msg.clear();
                    }

                    if (msg.empty()) {
                        std::cout << "브로드캐스트할 메시지가 비어 있습니다.\n";
                    }
                    else {
                        for (const auto& p : peers) {
                            int sent = ::sendto(
                                sockfd,
                                msg.c_str(),
                                msg.size(),
                                0,
                                reinterpret_cast<const sockaddr*>(&p.addr),
                                sizeof(p.addr)
                            );
                            if (sent < 0) {
                                perror("sendto() 실패");
                                break;
                            }
                        }
                        std::cout << "모든 피어에게 전송되었습니다.\n";
                    }
                }
                else {
                    std::cout << "알 수 없는 명령어입니다.\n";
                }

                printPrompt(currentTargetName);
        }

            // 일반 메시지: 현재 선택된 피어에게 전송
            if (!peers.empty()) {
                const Peer& target = peers[currentPeerIdx];
                int sent = ::sendto(
                    sockfd,
                    line.c_str(),
                    line.size(),
                    0,
                    reinterpret_cast<const sockaddr*>(&target.addr),
                    sizeof(target.addr)
                );

                if (sent < 0) {
                    perror("sendto() 실패");
                    break;
                }
            }
            else {
                std::cout << "등록된 피어가 없습니다. 메시지를 보낼 수 없습니다.\n";
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

            int received = ::recvfrom(
                sockfd,
                recv_buf,
                BUF_SIZE,
                0,
                reinterpret_cast<sockaddr*>(&from_addr),
                &from_len
            );

            if (received == SOCKET_ERROR) {
                int err = WSAGetLastError();
                cerr << "recvfrom() 실패, 오류 코드: " << err << endl;
                break;
            }

            recv_buf[received] = '\0';

            char from_ip[INET_ADDRSTRLEN];
            ::inet_ntop(AF_INET, &from_addr.sin_addr, from_ip, sizeof(from_ip));
            int from_port = ntohs(from_addr.sin_port);

            // 수신한 주소가 등록된 피어인지 확인
            std::string fromName = "Unknown";
            for (const auto& p : peers) {
                if (p.addr.sin_addr.s_addr == from_addr.sin_addr.s_addr &&
                    p.addr.sin_port == from_addr.sin_port) {
                    fromName = p.name;
                    break;
                }
            }

            std::cout << "\n[받음 " << fromName << " (" << from_ip << ":" << from_port
                << ")] " << recv_buf << "\n";

            printPrompt(currentTargetName);
        }
    }

    ::closesocket(sockfd);
    WSACleanup();
    return 0;
}
```

  
교수님께서 수정한 후 코드
```C++
// Mac/Linux용 코드에서 변경된 Windows 버전 (kbhit + select)

#include <iostream>
#include <string>
#include <cstring>
#include <limits>

#include <winsock2.h>      // [추가] WinSock2
#include <ws2tcpip.h>      // [추가] inet_pton, inet_ntop
#include <conio.h>         // [추가] _kbhit(), _getch()

#pragma comment(lib, "ws2_32.lib")   // [추가] ws2_32 라이브러리 링크

#define BUF_SIZE 1024

using namespace std;

int main() {
    SOCKET sockfd;                 // [변경] int → SOCKET
    sockaddr_in local_addr{};
    sockaddr_in peer_addr{};

    int local_port = 0;
    int peer_port = 0;
    string peer_ip_str;

    // [추가] WinSock 초기화
    WSADATA wsaData;
    int wsaRet = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (wsaRet != 0) {
        cerr << "WSAStartup 실패, 오류 코드: " << wsaRet << endl;
        return 1;
    }

    // ===============================
    // 1. 키보드로부터 설정 값 입력
    // ===============================
    cout << "[설정] 내(로컬) 포트 번호를 입력하세요: ";
    cin >> local_port;

    cout << "[설정] 상대방 IP 주소를 입력하세요: ";
    cin >> peer_ip_str;

    cout << "[설정] 상대방 포트 번호를 입력하세요: ";
    cin >> peer_port;

    // 남아 있는 개행 문자 제거
    cin.ignore((numeric_limits<streamsize>::max)(), '\n');

    if (local_port <= 0 || local_port > 65535 ||
        peer_port <= 0 || peer_port > 65535) {
        cerr << "포트 번호는 1~65535 사이여야 합니다.\n";
        WSACleanup();
        return 1;
    }

    // ===============================
    // 2. 소켓 생성 (UDP)
    // ===============================
    sockfd = ::socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == INVALID_SOCKET) {
        cerr << "socket() 실패, 오류 코드: " << WSAGetLastError() << endl;
        WSACleanup();
        return 1;
    }

    // ===============================
    // 3. 로컬 주소 설정 및 bind
    // ===============================
    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    local_addr.sin_port = htons(static_cast<u_short>(local_port)); // u_short 캐스팅

    if (::bind(sockfd, reinterpret_cast<sockaddr*>(&local_addr),
        sizeof(local_addr)) == SOCKET_ERROR) {
        cerr << "bind() 실패, 오류 코드: " << WSAGetLastError() << endl;
        ::closesocket(sockfd);
        WSACleanup();
        return 1;
    }

    // ===============================
    // 4. 상대방 주소 설정
    // ===============================
    memset(&peer_addr, 0, sizeof(peer_addr));
    peer_addr.sin_family = AF_INET;
    peer_addr.sin_port = htons(static_cast<u_short>(peer_port));

    if (::inet_pton(AF_INET, peer_ip_str.c_str(), &peer_addr.sin_addr) <= 0) {
        cerr << "잘못된 IP 주소: " << peer_ip_str << "\n";
        ::closesocket(sockfd);
        WSACleanup();
        return 1;
    }

    cout << "\n=== UDP 채팅 프로그램 (Windows + kbhit + select) 시작 ===\n";
    cout << "로컬 포트 : " << local_port << "\n";
    cout << "상대방    : " << peer_ip_str << ":" << peer_port << "\n";
    cout << "메시지를 입력하면 전송됩니다.\n";
    cout << "종료하려면 '/quit' 입력 후 Enter.\n\n";

    bool running = true;
    string inputBuffer;  // 한 줄 입력 버퍼

    cout << "> ";
    cout.flush();

    // ===============================
    // 5. select() + kbhit() 루프
    // ===============================
    while (running) {
        // 5-1. 소켓 수신 감시용 fd_set 구성
        fd_set readfds;
        FD_ZERO(&readfds);
        FD_SET(sockfd, &readfds);

        // Windows에서 select의 첫 번째 인자는 무시되므로 0으로 전달해도 됨
        TIMEVAL tv;
        tv.tv_sec = 0;
        tv.tv_usec = 100000;   // 0.1초 타임아웃

        int ret = ::select(0, &readfds, nullptr, nullptr, &tv);
        if (ret == SOCKET_ERROR) {
            int err = WSAGetLastError();
            cerr << "select() 실패, 오류 코드: " << err << endl;
            break;
        }

        // 5-2. 소켓에서 수신된 데이터 처리
        if (ret > 0 && FD_ISSET(sockfd, &readfds)) {
            char recv_buf[BUF_SIZE + 1];
            sockaddr_in from_addr{};
            int from_len = sizeof(from_addr);

            int received = ::recvfrom(
                sockfd,
                recv_buf,
                BUF_SIZE,
                0,
                reinterpret_cast<sockaddr*>(&from_addr),
                &from_len
            );

            if (received == SOCKET_ERROR) {
                int err = WSAGetLastError();
                cerr << "recvfrom() 실패, 오류 코드: " << err << endl;
                break;
            }

            recv_buf[received] = '\0';

            char from_ip[INET_ADDRSTRLEN];
            ::inet_ntop(AF_INET, &from_addr.sin_addr, from_ip, sizeof(from_ip));
            int from_port = ntohs(from_addr.sin_port);

            cout << "\n[받음 " << from_ip << ":" << from_port << "] "
                << recv_buf << "\n";

            // 다시 프롬프트 + 현재까지 입력한 내용 출력
            cout << "> " << inputBuffer;
            cout.flush();
        }

        // 5-3. 키보드 입력 확인 (한 글자씩 처리)
        while (_kbhit()) {
            int ch = _getch();   // 에코 없는 입력

            // 엔터(줄바꿈)
            if (ch == '\r' || ch == '\n') {
                cout << "\n";
                string line = inputBuffer;
                inputBuffer.clear();

                if (line == "/quit") {
                    cout << "채팅을 종료합니다.\n";
                    running = false;
                    break;
                }

                if (!line.empty()) {
                    int sent = ::sendto(
                        sockfd,
                        line.c_str(),
                        static_cast<int>(line.size()),
                        0,
                        reinterpret_cast<sockaddr*>(&peer_addr),
                        sizeof(peer_addr)
                    );

                    if (sent == SOCKET_ERROR) {
                        cerr << "sendto() 실패, 오류 코드: "
                            << WSAGetLastError() << endl;
                        running = false;
                        break;
                    }
                }

                cout << "> ";
                cout.flush();
            }
            // 백스페이스 처리
            else if (ch == 8 || ch == 127) {
                if (!inputBuffer.empty()) {
                    inputBuffer.pop_back();
                    cout << "\b \b";
                    cout.flush();
                }
            }
            // 그 외 일반 문자
            else {
                char c = static_cast<char>(ch);
                inputBuffer.push_back(c);
                cout << c;
                cout.flush();
            }
        }
    }

    ::closesocket(sockfd);
    WSACleanup();
    return 0;
}
```
