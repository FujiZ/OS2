#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <string.h>
#include <signal.h>
#include <fcntl.h>
#include <time.h>
#include "job.h"

int jobid=0;
int siginfo=1;
int fifo;
//int globalfd;//似乎并没有什么卵用

struct waitqueue *head[3]={NULL,NULL,NULL};//对应三个队列，数字越大优先级越高
int tslice[3]={5,2,1};//三个队列的时间片大小
struct waitqueue *next=NULL,*current =NULL;

struct timeval interval;
struct itimerval new,old;

/* 调度程序 */

void scheduler()
{
	struct jobinfo *newjob=NULL;
	struct jobcmd cmd;
	int  count = 0;
	bzero(&cmd,DATALEN);//初始化CMD
	if((count=read(fifo,&cmd,DATALEN))<0)
		error_sys("read fifo failed");
	#ifdef DEBUG
    	printf("Reading whether other process send command!\n");
		if(count){
			printf("cmd cmdtype\t%d\ncmd defpri\t%d\ncmd data\t%s\n",cmd.type,cmd.defpri,cmd.data);
		}
		else
			printf("no data read\n");
	#endif

	/* 更新等待队列中的作业 */
	#ifdef DEBUG
		printf("Update jobs in wait queue!\n");
    #endif
	updateall();

	switch(cmd.type){
	case ENQ:
	#ifdef DEBUG
		printf("Execute enq!\n");
    #endif
		do_enq(newjob,cmd);
		break;
	case DEQ:
	#ifdef DEBUG
		printf("Execute deq!\n");
    #endif
		do_deq(cmd);
		break;
	case STAT:
	#ifdef DEBUG
		printf("Execute stat!\n");
    #endif
		do_stat(cmd);
		break;
	default:
		break;
	}
	#ifdef DEBUG
		printf("Select which job to run next!\n");
	#endif
	/* 选择高优先级作业 */
	next=jobselect();
	/* 作业切换 */
	#ifdef DEBUG
		printf("Switch to the next job!\n");
	#endif
	jobswitch();
}
/*void scheduler()
{
	struct jobinfo *newjob=NULL;
	struct jobcmd cmd;
	int  count = 0;
	bzero(&cmd,DATALEN);//初始化CMD
	if((count=read(fifo,&cmd,DATALEN))<0)
		error_sys("read fifo failed");
	#ifdef DEBUG
    	printf("Reading whether other process send command!\n");
		if(count){
			printf("cmd cmdtype\t%d\ncmd defpri\t%d\ncmd data\t%s\n",cmd.type,cmd.defpri,cmd.data);
		}
		else
			printf("no data read\n");
	#endif

	//更新等待队列中的作业
	#ifdef DEBUG
		printf("Update jobs in wait queue!\n");
    #endif
	updateall();

	switch(cmd.type){
	case ENQ:
	#ifdef DEBUG
		printf("Execute enq!\n");
    #endif
		do_enq(newjob,cmd);
		break;
	case DEQ:
	#ifdef DEBUG
		printf("Execute deq!\n");
    #endif
		do_deq(cmd);
		break;
	case STAT:
	#ifdef DEBUG
		printf("Execute stat!\n");
    #endif
		do_stat(cmd);
		break;
	default:
		break;
	}
	#ifdef DEBUG
		printf("Select which job to run next!\n");
	#endif
	//选择高优先级作业
	next=jobselect();
	//作业切换
	#ifdef DEBUG
		printf("Switch to the next job!\n");
	#endif
	jobswitch();
}*/
int allocjid()
{
	return ++jobid;
}

void updateall()
{
	struct waitqueue *p,*prev;
	int i;
	#ifdef DEBUG
		printf("Before update:\n");
	#endif
	/* 更新作业运行时间 */
	if(current)
		current->job->run_time += interval.tv_sec; //加上上一个时间片的长度

	/* 更新作业等待时间及优先级 */
	for(i=2;i>=0;--i){
		for(prev=NULL,p = head[i]; p != NULL;){
			p->job->wait_time += interval.tv_sec*1000;
			if(p->job->wait_time >= 10000&i<2){//满足提高优先级条件
				p->job->curpri++;
				p->job->wait_time = 0;
				//将当前节点移动到高优先级队列队头
				if(p!=head[i]){
					prev->next=p->next;
					p->next=head[i+1];
					head[i+1]=p;
					p=prev->next;
				}
				else{
					head[i]=p->next;
					p->next=head[i+1];
					head[i+1]=p;
					p=head[i];
				}
			}
			else{
				prev=p;
				p=p->next;
			}
		}			
	}
	#ifdef DEBUG
		printf("After update:\n");
	#endif
}

struct waitqueue* jobselect()
{
	struct waitqueue *select;
	int i;
	select = NULL;
	
	for(i=2;i>=0;--i){
		if(head[i]){
			select=head[i];
			head[i]=select->next;
			return select;
		}
	}
	return select;
}

void jobswitch()
{
	struct waitqueue *p;
	int i;

	if(current && current->job->state == DONE){ /* 当前作业完成 */
		/* 作业完成，删除它 */
		for(i = 0;(current->job->cmdarg)[i] != NULL; i++){
			free((current->job->cmdarg)[i]);
			(current->job->cmdarg)[i] = NULL;
		}
		/* 释放空间 */
		free(current->job->cmdarg);
		free(current->job);
		free(current);

		current = NULL;
	}

	if(next == NULL && current == NULL){ /* 没有作业要运行 */
		interval.tv_sec=1;//设置时间片长度为默认
		return;
	}
	else if (next != NULL && current == NULL){ /* 开始新的作业 */
		printf("begin new job\n");
		current = next;
		next = NULL;
		current->job->state = RUNNING;
		kill(current->job->pid,SIGCONT);
		interval.tv_sec=tslice[current->job->curpri];//设置时间片长度
		return;
	}
	else if (next != NULL && current != NULL){ /* 切换作业 */

		printf("switch to Pid: %d\n",next->job->pid);
		kill(current->job->pid,SIGSTOP);
		//将当前作业的优先级降一级
		if(current->job->curpri>0)
			--current->job->curpri;
		current->job->wait_time = 0;
		current->job->state = READY;

		/* 放回等待队列 */
		if(head[current->job->curpri]){
			for(p = head[current->job->curpri]; p->next != NULL; p = p->next);
			p->next = current;
		}else{
			head[current->job->curpri] = current;
		}
		current = next;
		next = NULL;
		current->job->state = RUNNING;
		current->job->wait_time = 0;
		kill(current->job->pid,SIGCONT);
		interval.tv_sec=tslice[current->job->curpri];//设置时间片长度
		return;
	}else{ /* next == NULL且current != NULL，不切换 */
		return;
	}
}

void sig_handler(int sig,siginfo_t *info,void *notused)
{
	int status;
	int ret;

	switch (sig) {
case SIGVTALRM: /* 到达计时器所设置的计时间隔 */
	scheduler();
	#ifdef DEBUG
		printf("SIGVTALRM RECEIVED!\n");
    #endif
	//alarm(1);
	//每次执行完调度后重新设置时间
	new.it_value=interval;
	setitimer(ITIMER_VIRTUAL,&new,&old);
	return;
case SIGCHLD: /* 子进程结束时传送给父进程的信号 */
	ret = waitpid(-1,&status,WNOHANG);
	if (ret == 0)
		return;
	if(WIFEXITED(status)){
		current->job->state = DONE;
		printf("normal termation, exit status = %d\n",WEXITSTATUS(status));
	}else if (WIFSIGNALED(status)){
		printf("abnormal termation, signal number = %d\n",WTERMSIG(status));
	}else if (WIFSTOPPED(status)){
		printf("child stopped, signal number = %d\n",WSTOPSIG(status));
	}
	return;
	default:
		return;
	}
}

void do_enq(struct jobinfo *newjob,struct jobcmd enqcmd)
{
	struct waitqueue *newnode,*p;
	int i=0,pid;
	char *offset,*argvec,*q;
	char **arglist;
	sigset_t zeromask;

	sigemptyset(&zeromask);

	/* 封装jobinfo数据结构 */
	newjob = (struct jobinfo *)malloc(sizeof(struct jobinfo));
	newjob->jid = allocjid();
	newjob->defpri = enqcmd.defpri;
	newjob->curpri = enqcmd.defpri;
	newjob->ownerid = enqcmd.owner;
	newjob->state = READY;
	newjob->create_time = time(NULL);
	newjob->wait_time = 0;
	newjob->run_time = 0;
	arglist = (char**)malloc(sizeof(char*)*(enqcmd.argnum+1));
	newjob->cmdarg = arglist;
	offset = enqcmd.data;
	argvec = enqcmd.data;
	while (i < enqcmd.argnum){
		if(*offset == ':'){
			*offset++ = '\0';
			q = (char*)malloc(offset - argvec);
			strcpy(q,argvec);
			arglist[i++] = q;
			argvec = offset;
		}else
			offset++;
	}

	arglist[i] = NULL;

	#ifdef DEBUG

		printf("enqcmd argnum %d\n",enqcmd.argnum);
		for(i = 0;i < enqcmd.argnum; i++)
			printf("parse enqcmd:%s\n",arglist[i]);

	#endif

	/*向等待队列中增加新的作业*/
	newnode = (struct waitqueue*)malloc(sizeof(struct waitqueue));
	newnode->next =NULL;
	newnode->job=newjob;
	
	//将作业直接插到对应队列的队头
	newnode->next=head[newjob->curpri];
	head[newjob->curpri]=newnode;

	/*为作业创建进程*/
	if((pid=fork())<0)
		error_sys("enq fork failed");

	if(pid==0){
		newjob->pid =getpid();
		/*阻塞子进程,等等执行*/
		raise(SIGSTOP);
	#ifdef DEBUG

		printf("begin running\n");
		for(i=0;arglist[i]!=NULL;i++)
			printf("arglist %s\n",arglist[i]);
	#endif

		/*复制文件描述符到标准输出*/
		//dup2(globalfd,1);
		/* 执行命令 */
		if(execv(arglist[0],arglist)<0)
			printf("exec failed\n");
		exit(1);
	}else{
		newjob->pid=pid;
	}
	
}

void do_deq(struct jobcmd deqcmd)
{
	int deqid,i,j;
	struct waitqueue *p,*prev,*select,*selectprev;
	deqid=atoi(deqcmd.data);

#ifdef DEBUG
	printf("deq jid %d\n",deqid);
#endif

	/*current jodid==deqid,终止当前作业*/
	if (current && current->job->jid ==deqid){
		printf("teminate current job\n");
		kill(current->job->pid,SIGKILL);
		for(i=0;(current->job->cmdarg)[i]!=NULL;i++){
			free((current->job->cmdarg)[i]);
			(current->job->cmdarg)[i]=NULL;
		}
		free(current->job->cmdarg);
		free(current->job);
		free(current);
		current=NULL;
	}
	else{ /* 或者在等待队列中查找deqid */
		select=NULL;
		selectprev=NULL;
		for(i=2;i>=0;--i){
			if(head[i]){//当前队列非空
				for(prev=NULL,p=head[i];p!=NULL;prev=p,p=p->next){
					if(p->job->jid==deqid){
						select=p;
						selectprev=prev;
						break;
					}
				}
				if(select){
					if(select==head[i]){
						head[i]=select->next;
					}
					else{
						selectprev->next=select->next;
					}
					for(i=0;(select->job->cmdarg)[i]!=NULL;i++){
						free((select->job->cmdarg)[i]);
						(select->job->cmdarg)[i]=NULL;
					}
					free(select->job->cmdarg);
					free(select->job);
					free(select);
					select=NULL;
					return;
				}
			}
		}
	}
}

void do_stat(struct jobcmd statcmd)
{
	struct waitqueue *p;
	char timebuf[BUFLEN];
	/*
	*打印所有作业的统计信息:
	*1.作业ID
	*2.进程ID
	*3.作业所有者
	*4.作业运行时间
	*5.作业等待时间
	*6.作业创建时间
	*7.作业状态
	*/

	/* 打印信息头部 */
	printf("JOBID\tPID\tOWNER\tRUNTIME\tWAITTIME\tCREATTIME\t\tSTATE\n");
	if(current){
		strcpy(timebuf,ctime(&(current->job->create_time)));
		timebuf[strlen(timebuf)-1]='\0';
		printf("%d\t%d\t%d\t%d\t%d\t%s\t%s\n",
			current->job->jid,
			current->job->pid,
			current->job->ownerid,
			current->job->run_time,
			current->job->wait_time,
			timebuf,"RUNNING");
	}
	for(i=2;i>=0;--i){
		printf("In queue %d:\n",i);
		for(p=head[i];p!=NULL;p=p->next){
			strcpy(timebuf,ctime(&(p->job->create_time)));
			timebuf[strlen(timebuf)-1]='\0';
			printf("%d\t%d\t%d\t%d\t%d\t%s\t%s\n",
				p->job->jid,
				p->job->pid,
				p->job->ownerid,
				p->job->run_time,
				p->job->wait_time,
				timebuf,
				"READY");
		}
	}
}

int main()
{
	struct stat statbuf;
	struct sigaction newact,oldact1,oldact2;
    
	#ifdef DEBUG
		printf("DEBUG IS OPEN!\n");
    #endif
	if(stat("/tmp/server",&statbuf)==0){
		/* 如果FIFO文件存在,删掉 */
		if(remove("/tmp/server")<0)
			error_sys("remove failed");
	}

	if(mkfifo("/tmp/server",0666)<0)
		error_sys("mkfifo failed");
	/* 在非阻塞模式下打开FIFO */
	if((fifo=open("/tmp/server",O_RDONLY|O_NONBLOCK))<0)
		error_sys("open fifo failed");

	/* 建立信号处理函数 */
	newact.sa_sigaction=sig_handler;
	sigemptyset(&newact.sa_mask);
	newact.sa_flags=SA_SIGINFO;
	sigaction(SIGCHLD,&newact,&oldact1);
	sigaction(SIGVTALRM,&newact,&oldact2);

	/* 设置时间间隔为1000毫秒 */
	interval.tv_sec=1;
	interval.tv_usec=0;

	//new.it_interval=interval;
	new.it_value=interval;
	setitimer(ITIMER_VIRTUAL,&new,&old);
	//alarm(1);
	while(siginfo==1);

	close(fifo);
	//close(globalfd);
	return 0;
}
