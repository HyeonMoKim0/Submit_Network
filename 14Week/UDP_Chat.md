기말에서 Multi UDP와 그냥 UDP Chat 사이에 어떤 차이가 있는지 인지할 것  
그리고 select에서 굳이 타임아웃을 넣는 이유는, 없으면 윈도우에서 발생하는 이벤트를 처리할 틈이 없어짐  
그래서 TIMEVAL 이라는 구조체의 변수에 tv_usec = 100000;을 넣음으로써 0.1초의 타임아웃을 통해 이벤트를 처리하도록 조치한 것임  
사실상 타임아웃은 항상 있어야하는 꼴임, 다중으로 돌아가게 하려면.  

필자가 수정한 코드는 다음과 같음.
```C++
#include <iostream>
#include <string>
#include <cstring>
#include <limits>

// 기존 POSIX socket 관련 헤더는 삭제함

#include <winsock2.h>      // 윈속 초기화를 위함
#include <ws2tcpip.h>      // inet_pton, inet_ntop 사용 용도
#include <conio.h>         // _kbhit(), _getch() 사용 용도

#pragma comment(lib, "ws2_32.lib")   // ws2_32 라이브러리 링크

#define BUF_SIZE 1024

using namespace std;

int main() {
    SOCKET sockfd;                 // [변경] int → SOCKET, 유닉스에선 SOCKET이 int임
    sockaddr_in local_addr{};
    sockaddr_in peer_addr{};

    int local_port = 0;
    int peer_port = 0;
    string peer_ip_str;

    // 윈속 초기화
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
    cin.ignore(1, '\n');

    if (local_port <= 0 || local_port > 65535 ||
        peer_port <= 0 || peer_port > 65535) {
        cerr << "포트 번호는 1~65535 사이여야 합니다.\n";
        WSACleanup();
        return 1;
    }

    // ===============================
    // 2. 소켓 생성 (UDP)
    // ===============================
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == INVALID_SOCKET) { // 조건문 (sockfd < 0)에서 변경
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
    local_addr.sin_port = htons(static_cast<u_short>(local_port)); // uint16_t -> u_short

    if (bind(sockfd, reinterpret_cast<sockaddr*>(&local_addr),
        sizeof(local_addr)) == SOCKET_ERROR) { // 조건문 (bind(~~) < 0)에서 변경
        cerr << "bind() 실패, 오류 코드: " << WSAGetLastError() << endl;
        closesocket(sockfd); // 소켓이 생성되었으므로, 소켓도 닫음
        WSACleanup();
        return 1;
    }

    // ===============================
    // 4. 상대방 주소 설정
    // ===============================
    memset(&peer_addr, 0, sizeof(peer_addr));
    peer_addr.sin_family = AF_INET;
    peer_addr.sin_port = htons(static_cast<u_short>(peer_port)); // uint16_t -> u_short

    if (inet_pton(AF_INET, peer_ip_str.c_str(), &peer_addr.sin_addr) <= 0) {
        cerr << "잘못된 IP 주소: " << peer_ip_str << "\n";
        closesocket(sockfd);
        WSACleanup();
        return 1;
    }

    cout << "\n=== UDP 채팅 프로그램 시작 ===\n";
    cout << "로컬 포트 : " << local_port << "\n";
    cout << "상대방    : " << peer_ip_str << ":" << peer_port << "\n";
    cout << "메시지를 입력하면 전송됩니다.\n";
    cout << "종료하려면 '/quit' 입력 후 Enter.\n\n";

    bool running = true;

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

        int ret = select(0, &readfds, nullptr, nullptr, &tv);
        if (ret == SOCKET_ERROR) { // 조건문 (sockfd < 0)에서 변경
            cerr << "select() 실패, 오류 코드: " << WSAGetLastError() << endl; // 에러 코드 반환하도록 조정
            break;
        }

        // 5-2. 소켓에서 수신된 데이터 처리
        if (FD_ISSET(sockfd, &readfds)) {
            char recv_buf[BUF_SIZE + 1];
            struct sockaddr_in from_addr {};
            socklen_t from_len = sizeof(from_addr);

            int received = recvfrom( // ssize_t -> int
                sockfd,
                recv_buf,
                BUF_SIZE,
                0,
                reinterpret_cast<struct sockaddr*>(&from_addr),
                &from_len
            );

            if (received < 0) {
                perror("recvfrom() 실패");
                break;
            }

            recv_buf[received] = '\0';

            char from_ip[INET_ADDRSTRLEN];
            inet_ntop(AF_INET, &from_addr.sin_addr, from_ip, sizeof(from_ip));
            int from_port = ntohs(from_addr.sin_port);

            cout << "[받음 " << from_ip << ":" << from_port << "] "
                << recv_buf << "\n";
            cout.flush();
        }

        // 5-3. 키보드 입력 확인 (한 글자씩 처리)
        while (_kbhit()) {
            string line;
            if (!getline(cin, line)) {
                cout << "입력이 종료되었습니다. 프로그램을 종료합니다.\n";
                running = false; // 그냥 break만 있으면 키보드 입력 확인 반복문만 취소되므로, running 변수로 전체 반복문을 제어
                break;
            }

            if (line == "/quit") {
                cout << "채팅을 종료합니다.\n";
                running = false;
                break;
            }

            if (line.empty()) continue;

            int sent = sendto( // ssize_t -> int
                sockfd,
                line.c_str(),
                line.size(),
                0,
                reinterpret_cast<struct sockaddr*>(&peer_addr),
                sizeof(peer_addr)
            );
        }
    }

    closesocket(sockfd);
    WSACleanup();
    return 0;
}
```
