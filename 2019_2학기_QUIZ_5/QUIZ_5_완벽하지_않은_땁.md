- 문제 

  - 4명과 대화를 하는데 공유메모리와 세마포 이용해서 해결

- 완벽한 답은 아님 

  - 한명이 다른 사람 보낸 메세지를 느리게 받을 때, talk_quit 명령어를 입력해서 종료 시키면 종료되는 그 사람은 다른 사람에게 talk_quit 이라는 메세지를 안보내고 마저 못읽었던 메세지를 다 읽음처리해서 종료 시켜야함 <- 이 기능을 못함 (마저 못읽었던 메세지는 다읽고 종료를하는데, 다른 대화자들에게서 talk_quit이라는 메세지가 읽힘)

- 완벽하지 않은 답

  - ```c
    #include<sys/types.h>
    #include<sys/ipc.h>
    #include<sys/shm.h>
    #include<sys/sem.h>
    #include<sys/wait.h>
    #include<unistd.h>
    #include<stdio.h>
    #include<fcntl.h>
    #include<errno.h>
    #include<stdlib.h>
    #include<string.h>
    
    #define R 4
    #define B 5   // 실행예시 1 & 2 테스트 시는 10으로, 실행예시 3 테스트 시는 5로 설정
    #define L 512
    
    struct databuf1{
            int id;
    	int r[R];
    	char msg[L];
    };
    
    struct databuf2{
    	int write_pos;
    	int status_usr[4];
            struct databuf1 buf[B];
    };
    
    union semun{
            int val;
            struct semid_ds *buf;
            ushort *array;
    };
    
    void sem_wait(int semid, int semidx){
            struct sembuf p_buf;
    
            p_buf.sem_num=semidx;
            p_buf.sem_op=-1;
            p_buf.sem_flg=0;
            semop(semid, &p_buf, 1);
    }
    
    void sem_signal(int semid, int semidx){
            struct sembuf p_buf;
    
            p_buf.sem_num=semidx;
            p_buf.sem_op=1;
            p_buf.sem_flg=0;
            semop(semid, &p_buf, 1);
    }
    
    void reader(int N, int rindex, struct databuf2 *buf, int semid){
            int i, n, val;
    	char msg[L];	
    
            for (i=rindex; ; i=(i+1)%B){
    		if (N==1) sleep(5); // 실행예시 2 & 3 테스트 용
    		// 문자열 받기			
    		sem_wait(semid,N+3);
    
    		if(buf->buf[i].id != N && buf->status_usr[N] == 1){
    			printf("%d : %s\n",buf->buf[i].id+1, buf->buf[i].msg);
    		}
    
    		sem_wait(semid,2);
    		buf->buf[i].r[N] = 0;
    		
    		if(!buf->buf[i].r[0] && !buf->buf[i].r[1] && !buf->buf[i].r[2] && !buf->buf[i].r[3]){
    			sem_signal(semid,0);
    		}
    
    		sem_signal(semid,2);
    		
    		if(strcmp(buf->buf[i].msg,"talk_quit") == 0){
    			if(buf->buf[i].id == N){
    				exit(0);
    			}
    		}
    		
            }
    
            exit(0);
    }
    
    void writer(int N, struct databuf2 *buf, int semid){
            char temp[L];
            int i, n, flag=0;
    
            for ( ; ; ){
                    scanf("%s", temp);
    	
    		sem_wait(semid,0);
    		sem_wait(semid,1);
    		
    		n = buf->write_pos;
    		buf->write_pos = (n + 1) % B;
    		
    		buf->buf[n].id = N;
    		strcpy(buf->buf[n].msg,temp);
    		
    		sem_signal(semid,1);
    
    		sem_wait(semid,2);
    
    		for(i=0; i<4;i++){
    			if(buf->status_usr[i] == 1){
    				buf->buf[n].r[i] = 1;
    				sem_signal(semid,i+3);
    			}
    		}
    	
    		if(strcmp(temp,"talk_quit") == 0){
    			// sem_wait(semid,2);
    			buf->status_usr[N] = 0;
    			//sem_signal(semid,2);
    			break;
    		}
    		sem_signal(semid,2);
            }
    }
    
    int main(int argc, char** argv){
            int id, semid, shmid, i, j, N, rindex;
    	ushort init[7] = {B,1,1,0,0,0,0};
    	
            // init 선언
    	key_t key1, key2;
            union semun arg;
            struct databuf2 *buf;
    
    	// key1, key2 설정
    	key1 = ftok("./main.c",201);
    	key2 = ftok("./main.c",102);
    
    	id = atoi(argv[1]);
    
            shmid=shmget(key1, sizeof(struct databuf2), 0600 | IPC_CREAT);
    	buf = (struct databuf2 *)shmat(shmid,0,0);	
    
            if ((semid=semget(key2,7, 0600|IPC_CREAT|IPC_EXCL))>0){
                    arg.array= init;
                    semctl(semid, 0, SETALL, arg);
            }
            else{
                    semid=semget(key2,7, 0);
            }
    	
    	N = id-1;
    		
    	// 시작전 할일
    	sem_wait(semid,2);
    	rindex = buf->write_pos;
    	buf->status_usr[N] = 1;
    	sem_signal(semid,2);
    
    	for(i=0;i<4;i++){
    		printf(" %d ",buf->status_usr[i]);
    	}
    	printf("\n");
    
            if (fork()==0){
                    reader(N, rindex, buf, semid);
            }
            writer(N, buf, semid);
    
            wait(0);
    
            if (!buf->status_usr[0]  && !buf->status_usr[1] && !buf->status_usr[2] && !buf->status_usr[3]){
    		semctl(semid, IPC_RMID, 0);
    		shmctl(shmid, IPC_RMID, 0);
    	}
    
            exit(0);
    }
    
    ```

  - 