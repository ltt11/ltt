# ltt
#include<stdio.h>
#include<sys/socket.h>
#include<errno.h>
#include<ctype.h>
#include<netinet/in.h>
#include<netdb.h>
#include<string.h>
#include<arpa/inet.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<fcntl.h>
#define MAX 65535
#define THREAD_NUM 50
typedef struct IP_Message{
	int MinPort;
	int MaxPort;
	struct in_addr ip;
}IP_Message;
void *scan(void *arg){
	IP_Message *IP_M;
	char ipaddress[36];
    int i;
    int net;
    int err;
    struct hostent *host;
    struct sockaddr_in sa;
	int ret;
	fd_set wset;
	fd_set rset;
	int sel;//se:lect
	int flags;
	int get;//getsockopt
 
	int error=0;
	socklen_t len=sizeof(error);
 
	struct timeval tval;
	tval.tv_sec=1;
	tval.tv_usec=0;
 
	IP_M=arg;
    memset(&sa,0,sizeof(struct sockaddr_in));
    sa.sin_family=AF_INET;
    sa.sin_addr.s_addr=IP_M->ip.s_addr;
 
 
 
    for(i=1;i<1025;i++){
        sa.sin_port=htons(i);
  		if((net=socket(AF_INET,SOCK_STREAM,0))<0){
    		 perror("\nsocket");
       		 exit(2);
   		}
 
 
		flags=fcntl(net,F_GETFL,0);//鑾峰彇鏂囦欢鐨刦lags鍊?
		fcntl(net,F_SETFL,flags|O_NONBLOCK);//璁剧疆鎴愰潪闃诲妯″紡
 
		ret=connect(net,(struct sockaddr*)&sa,sizeof(sa));
 
		if(ret==0){
			printf("%s   %-5d\n",inet_ntoa(sa.sin_addr),i);
			close(net);
		}
		else{
				if(errno==EINPROGRESS){
					FD_ZERO(&wset);
					FD_ZERO(&rset);
					FD_SET(net,&wset);
					FD_SET(net,&rset);
					sel=select(net+1,&rset,&wset,NULL,&tval);
					if(sel<=0||sel==2){
						close(net);
					}
					else if(sel==1&&FD_ISSET(net,&wset)){
								printf("%s   %-5d Opened!\n",inet_ntoa(sa.sin_addr),i);
							close(net);
					}
					else{
							close(net);
					}
				}
		}
	}
}
void mul_scan(char *arg){
	int i;
	struct hostent *he;
	printf("%s :",arg);
	pthread_t worker_tid;
	IP_Message IP;
	IP.ip.s_addr=inet_addr(arg);
	he=gethostbyaddr((char *)&IP.ip.s_addr,sizeof(struct in_addr),AF_INET);
	if(he!=NULL)
		printf("%s\n",he->h_name);	
	else
		printf("\n");
   		if(pthread_create(&worker_tid,NULL,scan,(void *)&IP)!=0){
       		perror("create:");
       		exit(1);
		}
	pthread_join(worker_tid,NULL);
}
int main(){
	char ip[16];
	char buffer[100];
	FILE *fp;	
	fp=fopen("/proc/net/arp","r");
	if(!fp){
			perror("fp");
			exit(2);
	}
	fgets(buffer,100,fp);
	while(!feof(fp)){
		fgets(ip,16,fp);
		if(feof(fp))
				break;
		mul_scan(ip);
		fgets(buffer,100,fp);
	}
	fclose(fp);
    return 0;
}

