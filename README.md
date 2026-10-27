# hammingCode-
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
int recvall(int s, void *buf, int len) {
 int t = 0, n;
 while (t < len) {
 n = recv(s, (char *)buf + t, len - t, 0);
 if (n <= 0) return -1;
 t += n;
 }
 return 0;
}
int main(void) {
 int s, ns, i, j, p, n, error = 0, c, len;
 char code[200], bits[200] = {0}, data[200];
 struct sockaddr_in server;
 s = socket(AF_INET, SOCK_STREAM, 0);
 if (s < 0) { perror("socket"); return 1; }
 int opt = 1;
 if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
 perror("setsockopt");
 close(s);
 return 1;
 }
 memset(&server, 0, sizeof(server));
 server.sin_family = AF_INET;
 server.sin_port = htons(5000);
 server.sin_addr.s_addr = INADDR_ANY;
 if (bind(s, (struct sockaddr *)&server, sizeof(server)) < 0) {
 perror("bind"); close(s); return 1;
 }
 if (listen(s, 1) < 0) {
 perror("listen"); close(s); return 1;
 }
 printf("Waiting for sender...\n");
 ns = accept(s, NULL, NULL);
 if (ns < 0) {
 perror("accept"); close(s); return 1;
 }
 if (recvall(ns, &len, sizeof(len)) < 0) {
 printf("Failed to receive length.\n");
 close(ns); close(s); return 1;
 }
 n = ntohl(len);
 if (n <= 0 || n >= 200) {
 printf("Invalid code length.\n");
 close(ns); close(s); return 1;
 }
 if (recvall(ns, code, n) < 0) {
 printf("Failed to receive code.\n");
 close(ns); close(s); return 1;
 }
 code[n] = '\0';
 for (i = 0; i < n; i++) {
 if (code[i] != '0' && code[i] != '1') {
 printf("Invalid received data.\n");
 close(ns); close(s); return 1;
 }
 }
 printf("Received code: %s\n", code);
 for (i = 1; i <= n; i++)
 bits[i] = code[n - i] - '0'; Hamming Code over TCP sockets Page 2
 for (p = 1; p <= n; p *= 2) {
 c = 0;
 for (i = 1; i <= n; i++)
 if (i & p) c ^= bits[i];
 if (c) error += p;
 }
 if (error > 0 && error <= n) {
 printf("Error at position %d\n", error);
 bits[error] ^= 1;
 } else if (error > n) {
 printf("Invalid syndrome.\n");
 close(ns); close(s); return 1;
 } else {
 printf("No error detected.\n");
 }
 printf("Corrected code: ");
 for (i = n; i >= 1; i--) printf("%d", bits[i]);
 printf("\n");
 j = 0;
 for (i = n; i >= 1; i--)
 if (i & (i - 1)) data[j++] = bits[i] + '0';
 data[j] = '\0';
 printf("Original data: %s\n", data);
 close(ns);
 close(s);
 return 0;
}


sender code

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
int sendall(int s, const void *buf, int len) {
 int t = 0, n;
 while (t < len) {
 n = send(s, (const char *)buf + t, len - t, 0);
 if (n <= 0) return -1;
 t += n;
 }
 return 0;
}
int main(void) {
 int s, i, j, m, r = 0, n, c;
 char data[100], code[200], bits[200] = {0};
 struct sockaddr_in server;
 printf("Enter binary data: ");
 if (scanf("%99s", data) != 1) return 1;
 m = strlen(data);
 for (i = 0; i < m; i++) {
 if (data[i] != '0' && data[i] != '1') {
 printf("Invalid binary data.\n");
 return 1;
 }
 }
 while ((1 << r) < m + r + 1) r++;
 n = m + r;
 j = m - 1;
 for (i = 1; i <= n; i++)
 if (i & (i - 1)) bits[i] = data[j--] - '0';
 for (i = 1; i <= n; i *= 2) {
 c = 0;
 for (j = 1; j <= n; j++)
 if ((j & i) && j != i) c ^= bits[j];
 bits[i] = c;
 }
 for (i = 1; i <= n; i++)
 code[n - i] = bits[i] + '0';
 code[n] = '\0';
 printf("Hamming code: %s\n", code);
 s = socket(AF_INET, SOCK_STREAM, 0);
 if (s < 0) { perror("socket"); return 1; }
 memset(&server, 0, sizeof(server));
 server.sin_family = AF_INET;
 server.sin_port = htons(5000);
 server.sin_addr.s_addr = inet_addr("127.0.0.1");
 if (connect(s, (struct sockaddr *)&server, sizeof(server)) < 0) {
 perror("connect");
 close(s);
 return 1;
 }
 int len = htonl(n);
 if (sendall(s, &len, sizeof(len)) < 0 ||
 sendall(s, code, n) < 0) {
 perror("send");
 close(s);
 return 1;
 }
 printf("Data sent successfully.\n");
 close(s);
 return 0; Hamming Code over TCP sockets Page 2
}