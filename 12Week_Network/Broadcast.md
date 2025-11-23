다음은 Receiver의 브로드캐스트 코드다.
```C++
#include "..\..\Common.h"

#define LOCALPORT 9000
#define BUFSIZE   512

int main(int argc, char* argv[])
{
	int retval;

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET sock = socket(AF_INET, SOCK_DGRAM, 0);
	if (sock == INVALID_SOCKET) err_quit("socket()");

	// Sender에 있던 브로드캐스팅 활성화 코드 적용, 대신 SO_BREOADCAST 대신 다른거 사용
	DWORD bEnable = 1;
	retval = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, //소켓 옵션 설정, 
		(const char*)&bEnable, sizeof(bEnable));
	if (retval == SOCKET_ERROR) err_quit("setsockopt()");

	// bind()
	struct sockaddr_in localaddr;
	memset(&localaddr, 0, sizeof(localaddr));
	localaddr.sin_family = AF_INET;
	localaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	localaddr.sin_port = htons(LOCALPORT);
	retval = bind(sock, (struct sockaddr*)&localaddr, sizeof(localaddr));
	if (retval == SOCKET_ERROR) err_quit("bind()"); //동일한 소켓 주소 쓰면 두 개의 Receiver 중 한 녀석 오류 발생

	// 데이터 통신에 사용할 변수
	struct sockaddr_in peeraddr;
	int addrlen;
	char buf[BUFSIZE + 1];

	// 브로드캐스트 데이터 받기
	while (1) {
		// 데이터 받기
		addrlen = sizeof(peeraddr);
		retval = recvfrom(sock, buf, BUFSIZE, 0,
			(struct sockaddr*)&peeraddr, &addrlen);
		if (retval == SOCKET_ERROR) {
			err_display("recvfrom()");
			break;
		}

		// 받은 데이터 출력
		buf[retval] = '\0';
		char addr[INET_ADDRSTRLEN];
		inet_ntop(AF_INET, &peeraddr.sin_addr, addr, sizeof(addr));
		//printf("[UDP/%s:%d] %s\n", addr, ntohs(peeraddr.sin_port), buf);
		printf("[UDP/%s:%d] %s%d\n", addr, ntohs(peeraddr.sin_port), buf, (int)buf[strlen(buf) - 1]);
	}

	// 소켓 닫기
	closesocket(sock);

	// 윈속 종료
	WSACleanup();
	return 0;
}
```

다음은 Sender의 브로드캐스트 코드다.
```C++
#include "..\..\Common.h"

#define REMOTEIP   "255.255.255.255"
//#define REMOTEIP   "113.198.241.255" //자신의 IP 주소에 마지막 네번째 숫자에 255를 붙이면 원래는 잘 전송돼야 함
#define REMOTEPORT 9000
#define BUFSIZE    512

int main(int argc, char* argv[])
{
	int retval;

	// 윈속 초기화
	WSADATA wsa;
	if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
		return 1;

	// 소켓 생성
	SOCKET sock = socket(AF_INET, SOCK_DGRAM, 0);
	if (sock == INVALID_SOCKET) err_quit("socket()");

	// 브로드캐스팅 활성화
	DWORD bEnable = 1; //원래 1인데 0으로 한다면, 브로드캐스트를 안 하겠다는 의미임
	retval = setsockopt(sock, SOL_SOCKET, SO_BROADCAST, //소켓 옵션 설정, 
		(const char*)&bEnable, sizeof(bEnable));
	if (retval == SOCKET_ERROR) err_quit("setsockopt()");

	// 소켓 주소 구조체 초기화
	struct sockaddr_in remoteaddr;
	memset(&remoteaddr, 0, sizeof(remoteaddr));
	remoteaddr.sin_family = AF_INET;
	inet_pton(AF_INET, REMOTEIP, &remoteaddr.sin_addr);
	remoteaddr.sin_port = htons(REMOTEPORT);

	// 데이터 통신에 사용할 변수
	char buf[BUFSIZE + 1];
	int len;

	int count = 1; //메시지 보낸 횟수
	char countVal[sizeof(int)]; //메시지 보낸 횟수를 담는 배열

	// 브로드캐스트 데이터 보내기
	while (1) {
		// 데이터 입력
		printf("\n[보낼 데이터] ");
		if (fgets(buf, BUFSIZE + 1, stdin) == NULL)
			break;

		//// '\n' 문자 제거
		//len = (int)strlen(buf);
		//if (buf[len - 1] == '\n')
		//	buf[len - 1] = '\0';
		//if (strlen(buf) == 0)
		//	break;

		// 끝문자의 별도 처리
		len = (int)strlen(buf);
		buf[len - 1] = ' ';	// 개행문자 제거 및 구분자 삽입
		buf[len] = '-';		// 구분자 삽입
		buf[len + 1] = ' ';	// 구분자 삽입
		
		memcpy(countVal, &count, sizeof(count)); // 안전하게 count값 복사
		for (int i = 0; i < sizeof(int); ++i)	 // 각각의 바이트를 buf에 삽입
			buf[len + 2 + i] = countVal[i];

		len = (int)strlen(buf); // 새로운 길이 계산
		buf[len] = '\0'; // 문자열 끝에 NULL 문자 삽입
		
		count++; // 보낸 메시지 횟수 증가

		// 데이터 보내기
		retval = sendto(sock, buf, (int)strlen(buf), 0,
			(struct sockaddr*)&remoteaddr, sizeof(remoteaddr));
		if (retval == SOCKET_ERROR) {
			err_display("sendto()");
			break;
		}
		printf("[UDP] %d바이트를 보냈습니다.\n", retval);
	}

	// 소켓 닫기
	closesocket(sock);

	// 윈속 종료
	WSACleanup();
	return 0;
}
```
