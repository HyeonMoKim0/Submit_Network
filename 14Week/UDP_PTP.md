```C++
#include "..\..\Common.h"
#include <string>
#include <iostream>
#include <iterator>
#include <map>
#include <thread>

#define _WINSOCK_DEPRECATED_NO_WARNINGS  // inet_ntoa 등 구버전 함수 경고 무시(예제용)
#include <winsock2.h>
#include <ws2tcpip.h>

using namespace std;

//char* SERVERIP = (char*)"127.0.0.1";
char* SERVERIP = (char*)"192.0.0.1";
#define SERVERPORT 9000
#define BUFSIZE    512

SOCKET sock;

#define BACKLOG   10     // listen 대기 큐 크기

void RecvThread() {
	struct sockaddr_in peeraddr;
	int addrlen;
	char buf2[BUFSIZE + 1];
	int retval;

	while (1) {
		addrlen = sizeof(peeraddr);
		// 여기서 블로킹되어도 메인 스레드(입력)에는 영향 없음
		retval = recvfrom(sock, buf2, BUFSIZE, 0, (struct sockaddr*)&peeraddr, &addrlen);
		
		if (retval == SOCKET_ERROR) { continue; }
		buf2[retval] = '\0';
		printf("\n[메시지 수신]: %s\n[입력]: ", buf2);
	}
}

int main(int argc, char* argv[])
{
	int retval;

	string cName = "김현모";
	map<string, char*> addr;
	//addr.insert(pair<string, char*>({ "김현모", (char*)"113.198.243.168" }));
	//addr.insert(pair<string, char*>({ "아무개", (char*)"113.198.243.168" }));
	//addr.insert(pair<string, char*>({ "김현모", (char*)"222.97.255.192" }));
	//addr.insert(pair<string, char*>({ "김현모", (char*)"192.168.50.248" }));
	addr.insert(pair<string, char*>({ "김현모", (char*)"192.168.50.188" }));

	addr.insert(pair<string, char*>({ "김현준", (char*)"113.198.243.182" }));
	addr.insert(pair<string, char*>({ "김덕용", (char*)"113.198.243.193" }));
	addr.insert(pair<string, char*>({ "김도한", (char*)"113.198.243.187" }));
	addr.insert(pair<string, char*>({ "이성민", (char*)"113.198.243.167" }));

	// 명령행 인수가 있으면 IP 주소로 사용
	if (argc > 1) SERVERIP = argv[1];

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	sock = socket(AF_INET, SOCK_DGRAM, 0);
	if (sock == INVALID_SOCKET) err_quit("socket()");

	// 자신의 포트 번호 고정
	struct sockaddr_in localaddr;
	memset(&localaddr, 0, sizeof(localaddr));
	localaddr.sin_family = AF_INET;
	localaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	localaddr.sin_port = htons(SERVERPORT);
	retval = bind(sock, (struct sockaddr*)&localaddr, sizeof(localaddr));
	if (retval == SOCKET_ERROR) {
		printf("[오류] bind() 실패\n");
		return 1;
	}

	// 통신 서버 변경
	cout << "누구에게 보내시겠습니까?" << endl;
	for (pair<string, char*> a : addr) { cout << a.first << endl; }
	cout << "[입력] : ";
	cin >> cName;

	map<string, char*>::iterator it = find_if(addr.begin(), addr.end(), [cName](pair<string, char*> a) { return a.first == cName; });
	cout << it->second << endl;
	SERVERIP = it->second;

	// 소켓 주소 구조체 초기화
	struct sockaddr_in serveraddr;
	memset(&serveraddr, 0, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	inet_pton(AF_INET, SERVERIP, &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);


	// 데이터 통신에 사용할 변수
	struct sockaddr_in peeraddr;
	int addrlen;
	char buf[BUFSIZE + 1] = {}; //sendto 용도
	//char buf2[BUFSIZE + 1] = {}; //recvfrom 용도
	int len;

	cin.ignore(1, '\n');

	thread recv_thread(RecvThread);
	recv_thread.detach(); // 메인 스레드와 분리하여 독립적으로 실행

	// 서버와 데이터 통신
	while (1) {
		// 데이터 입력
		if (fgets(buf, BUFSIZE + 1, stdin) == NULL)
			break;

		// '\n' 문자 제거
		len = (int)strlen(buf);
		if (buf[len - 1] == '\n')
			buf[len - 1] = '\0';
		if (strlen(buf) == 0)
			break;

		// 데이터 보내기
		retval = sendto(sock, buf, (int)strlen(buf), 0,
			(struct sockaddr*)&serveraddr, sizeof(serveraddr));
		if (retval == SOCKET_ERROR) {
			err_display("sendto()");
			break;
		}
		printf("[UDP 클라이언트] %d바이트를 보냈습니다.\n", retval);
	}

	// 소켓 닫기
	closesocket(sock);

	// 윈속 종료
	WSACleanup();
	return 0;
}
```
