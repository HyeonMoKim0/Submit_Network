다음은 서버의 네트워크 코드다.

```C++
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include "..\..\Common.h"
#include <iostream>
#include <fstream>

#define SERVERPORT 9000
#define BUFSIZE    512

int main(int argc, char *argv[])
{
	int retval;

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
	if (listen_sock == INVALID_SOCKET) err_quit("socket()");

	// bind()
	struct sockaddr_in serveraddr;
	memset(&serveraddr, 0, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
	serveraddr.sin_port = htons(SERVERPORT);
	retval = bind(listen_sock, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
	if (retval == SOCKET_ERROR) err_quit("bind()");

	// listen()
	retval = listen(listen_sock, SOMAXCONN);
	std::cout << "[서버] 클라이언트 접속 대기 중...\n";
	if (retval == SOCKET_ERROR) err_quit("listen()");

	// 데이터 통신에 사용할 변수
	SOCKET client_sock;
	struct sockaddr_in clientaddr;
	int addrlen;
	char buf[BUFSIZE + 1];

	while (1) {
		// accept()
		addrlen = sizeof(clientaddr);
		client_sock = accept(listen_sock, (struct sockaddr *)&clientaddr, &addrlen);
		if (client_sock == INVALID_SOCKET) {
			err_display("accept()");
			break;
		}

		// 접속한 클라이언트 정보 출력
		char addr[INET_ADDRSTRLEN];
		inet_ntop(AF_INET, &clientaddr.sin_addr, addr, sizeof(addr));
		printf("\n[TCP 서버] 클라이언트 접속: IP 주소=%s, 포트 번호=%d\n",
			addr, ntohs(clientaddr.sin_port));

		// 클라이언트와 데이터 통신
		while (1) {
			// 데이터 받기
			retval = recv(client_sock, buf, BUFSIZE, 0);
			if (retval == SOCKET_ERROR) {
				err_display("recv()");
				break;
			}
			else if (retval == 0)
				break;

			// 받은 데이터 출력
			buf[retval] = '\0';
			printf("[TCP/%s:%d] %s\n", addr, ntohs(clientaddr.sin_port), buf);
			
			if (!strcmp(buf, "list") || !strcmp(buf, "get test.txt") || !strcmp(buf, "get pioneer.png")) {
				if (!strcmp(buf, "list")) {
					char fileList[] = "test.txt\npioneer.png\n";
					send(client_sock, fileList, strlen(fileList), 0);
				}
				else if (!strcmp(buf, "get test.txt")) {
					std::ifstream file("test.txt", std::ios::binary);
					if (!file) {
						std::cerr << "[서버] test.txt 파일을 열 수 없습니다.\n";
						closesocket(client_sock);
						closesocket(listen_sock);
						WSACleanup();
						return 1;
					}

					char buffer[1024];
					while (!file.eof()) {
						file.read(buffer, sizeof(buffer));
						int bytesRead = file.gcount();
						send(client_sock, buffer, bytesRead, 0);
					}

					// 파일의 마지막 조각 처리
					int bytesRead = file.gcount();
					if (bytesRead > 0) {
						send(client_sock, buffer, bytesRead, 0);

						std::cout << "[서버] 파일 전송 완료!\n";
						file.close();
					}
				}
				else if (!strcmp(buf, "get pioneer.png")) {
					std::ifstream file("pioneer.png", std::ios::binary);
					if (!file) {
						std::cerr << "[서버] pioneer.png 파일을 열 수 없습니다.\n";
						closesocket(client_sock);
						closesocket(listen_sock);
						WSACleanup();
						return 1;
					}

					char buffer[1024];
					while (!file.eof()) {
						file.read(buffer, sizeof(buffer));
						int bytesRead = file.gcount();
						send(client_sock, buffer, bytesRead, 0);
					}

					// 파일의 마지막 조각 처리
					int bytesRead = file.gcount();
					if (bytesRead > 0) {
						send(client_sock, buffer, bytesRead, 0);

						std::cout << "[서버] 파일 전송 완료!\n";
						file.close();
					}
				}
			}
			else {
				// 데이터 보내기
				retval = send(client_sock, buf, retval, 0);
				if (retval == SOCKET_ERROR) {
					err_display("send()");
					break;
				}
			}
		}

		// 소켓 닫기
		closesocket(client_sock);
		printf("[TCP 서버] 클라이언트 종료: IP 주소=%s, 포트 번호=%d\n",
			addr, ntohs(clientaddr.sin_port));
	}

	// 정리
	closesocket(listen_sock);
	
	WSACleanup();
	return 0;
}
```

다음은 네트워크 클라이언트 코드다.
```C++
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include "..\..\Common.h"
#include <iostream>
#include <fstream>

char *SERVERIP = (char *)"127.0.0.1";
#define SERVERPORT 9000
#define BUFSIZE    512

int main(int argc, char *argv[]) {
	int retval;

	// 명령행 인수가 있으면 IP 주소로 사용
	if (argc > 1) SERVERIP = argv[1];

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);
	if (sock == INVALID_SOCKET) err_quit("socket()");

	// connect()
	struct sockaddr_in serveraddr;
	memset(&serveraddr, 0, sizeof(serveraddr));
	serveraddr.sin_family = AF_INET;
	inet_pton(AF_INET, SERVERIP, &serveraddr.sin_addr);
	serveraddr.sin_port = htons(SERVERPORT);
	retval = connect(sock, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
	if (retval == SOCKET_ERROR) err_quit("connect()");

	// 데이터 통신에 사용할 변수
	char buf[BUFSIZE + 1];
	int len;

	// 서버와 데이터 통신
	while (1) {
		// 데이터 입력
		printf("\n[보낼 데이터] ");
		if (fgets(buf, BUFSIZE + 1, stdin) == NULL)
			break;

		// '\n' 문자 제거
		len = (int)strlen(buf);
		if (buf[len - 1] == '\n')
			buf[len - 1] = '\0';
		if (strlen(buf) == 0)
			break;

		// 데이터 보내기
		retval = send(sock, buf, (int)strlen(buf), 0);
		buf[retval] = '\0';
		if (retval == SOCKET_ERROR) {
			err_display("send()");
			break;
		}
		printf("[TCP 클라이언트] %d바이트를 보냈습니다.\n", retval);

		if (!strcmp(buf, "list") || !strcmp(buf, "get test.txt") || !strcmp(buf, "get pioneer.png")) {
			if (!strcmp(buf, "list")) {
				retval = recv(sock, buf, sizeof(buf), 0); //22: "test.txt\npioneer.png\n"

				buf[retval] = '\0';
				printf(buf);
			}
			else if (!strcmp(buf, "get test.txt")) {
				std::ofstream outFile("received.txt", std::ios::binary);
				char buffer[1024];
				int bytesReceived;

				while ((bytesReceived = recv(sock, buffer, sizeof(buffer), 0)) > 0)
					outFile.write(buffer, bytesReceived);

				outFile.close();
			}
			else if (!strcmp(buf, "get pioneer.png")) {
				std::ofstream outFile("pioneer.png", std::ios::binary);
				char buffer[1024];
				int bytesReceived;

				while ((bytesReceived = recv(sock, buffer, sizeof(buffer), 0)) > 0)
					outFile.write(buffer, bytesReceived);

				outFile.close();
			}
		}
		else {
			// 데이터 받기
			retval = recv(sock, buf, retval, 0);

			// 받은 데이터 출력
			buf[retval] = '\0';
			printf("[TCP 클라이언트] %d바이트를 받았습니다.\n", retval);
			printf("[받은 데이터] %s\n", buf);
		}
	}

	// 정리
	closesocket(sock);
	WSACleanup();
	return 0;
}
```
