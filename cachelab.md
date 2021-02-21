<h1>cachelab</h1>

​		cachelab难度明显又超过前面几个lab了，有挑战才会有进步。

<h2>前置知识</h2>

<h3>CPU，主存，磁盘</h3>

![image-20210211174717519](C:\Users\wolf\AppData\Roaming\Typora\typora-user-images\image-20210211174717519.png)

​		这是CPU从主存中读取数据的主要流程。计算机中的存储系统是分层的，不同层级间的数据访问速度差异巨大。

​		CPU中算数逻辑单元直接从寄存器中存取数据进行运算，随机存取存储器有两种，SRAM和DRAM，SRAM非常快，通常用在处理器做缓存，但是比较贵。DRAM慢一些，需要刷新，通常用作主存，相对来说比SRAM便宜。

​		SRAM和DRAM之中保留的数据，断电之后不会保存，要保存数据需要磁盘，常见的有固态硬盘和机械硬盘。

​		总线是用来传输地址、数据和控制信号的一组平行的电线，通常来说由多个设备共享，类似于不同城市之间的高速公路，可以传输各类数据。CPU 通过总线和对应的接口来从不同的设备中获得所需要的数据，放入寄存器中等待运算。

​		假设 CPU 需要从硬盘中读取一些数据，会给定指令，逻辑块编号和目标地址，并发送给磁盘控制器。然后磁盘控制器会读取对应的数据，并通过 DMA(direct memory access)把数据传输到内存中；传输完成后，磁盘控制器通过中断的方式通知 CPU，然后 CPU 完成之后的工作。

<h3>存储体系 Memory Hierarchy</h3>

​		CPU，主存以及本地磁盘之间的数据存取和访问速度差异越来越大。所以，利用局部性特性，设计出了分层式的金字塔式存储体系，通过高速缓存，来解决CPU和内存之间访问速度差异的问题。

![image-20210211182537198](C:\Users\wolf\AppData\Roaming\Typora\typora-user-images\image-20210211182537198.png)

<h4>局部性原理</h4>

<h5>时间局部性</h5>

​		如果一个信息项正在被访问，那么在一定时间内他会再次被访问。

<h5>空间局部性</h5>

​		在最近的将来用到的信息可能与现在正在使用的信息在空间地址上是临近的。

<pre>
    通过遍历多维数组的过程中，可以很好的理解局部性的概念。
</pre>

<h3>高速缓冲存储器 Cache Memory</h3>

<h4>基本结构</h4>

​		Cache的基本机构如下所示：![img](https://wdxtub.com/images/14612628563878.jpg)

​		集合中不只有一行，上面这个是直接映射高速缓存的图示。

![img](https://wdxtub.com/images/14612633441722.jpg)

​		标记位中，s用来匹配集合(set)，t用来匹配行，b用来匹配行中的偏移。

![img](https://wdxtub.com/images/14612642281687.jpg)

<h2>PART A</h2>

​		目的是实现一个缓存模拟器，只要输出命中和不命中，不需要存储数据。

```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<getopt.h>
#include<math.h>
#include "cachelab.h"

typedef struct	Cache{
	int valid,tag,stamp;
}cache_line;

char filename[0x100];
// set v_mode static
int v_mode;
int offset;
int s;
int lines;
int hit_count=0;
int miss_count=0;
int eviction_count=0;

void help()
{
	puts("Usage: ./csim [-hv] -s <num> -E <num> -b <num> -t <file>");
	puts("Options:");
	puts("  -h         Print this help message.");
	puts("  -v         Optional verbose flag.");
	puts("  -s <num>   Number of set index bits.");
	puts("  -E <num>   Number of lines per set.");
	puts("  -b <num>   Number of block offset bits.");
	puts("  -t <file>  Trace file.\n");
	puts("Examples:");
	puts("  linux>  ./csim -s 4 -E 1 -b 4 -t traces/yi.trace");
	puts("  linux>  ./csim -v -s 8 -E 2 -b 4 -t traces/yi.trace");
}

void initData(cache_line** Mycache){
	int j, k,S;
	S=1<<s;
	for (j = 0;j < S;j++) {
		for (k = 0; k < lines; k++) {
			Mycache[j][k].valid=0;
			Mycache[j][k].tag=0;
			Mycache[j][k].stamp=0;
		}
	}
}

void accessData(cache_line** Mycache,int lines,unsigned int address){
	int i;
	int max_stamp=0;
	int max_stamp_id=-1;
	int target_set = (address >> offset) & ((-1U) >> (32-s));
	int t_address = address >> (s+offset);
	for(i=0;i<lines;i++){
		if(Mycache[target_set][i].valid==0 && Mycache[target_set][i].tag==0){
			miss_count++;
			Mycache[target_set][i].valid==1;
			Mycache[target_set][i].tag=t_address;
			Mycache[target_set][i].stamp=1;
			if(v_mode){	printf("%s","miss");}
			return ;
		}
	}
	for(i=0;i<lines;i++){	
		if(Mycache[target_set][i].tag==t_address){
			Mycache[target_set][i].stamp=1;
			if(v_mode){	printf("%s","hit");}
			return ;
		}
	}

	eviction_count++;
	miss_count++;
	for(i=0;i<lines;i++){
		if(Mycache[target_set][i].stamp > max_stamp)
		{
			max_stamp=Mycache[target_set][i].stamp;
			max_stamp_id=i;
		}
	}
	Mycache[target_set][max_stamp_id].tag=t_address;
	Mycache[target_set][max_stamp_id].stamp=0;
	if(v_mode)
		printf("%s","miss hit");
	return ;
}

void simulator(cache_line** Mycache,char* filename,int S,int lines){
	char identifier;
	unsigned int address;
	int size,m,n,i;
	unsigned int count;
	FILE *fp=fopen(filename,"r");
	if(fp==NULL){
		fprintf(stderr,"Open Error.\n");
		exit(-1);
	}
	while(fscanf(fp,"%c %x,%d",&identifier,&address,&size)!=-1){
		count=S*lines;
        	// check identifier
        	m=(address%count)/lines;
        	n=(address%count)%lines;
		switch(identifier){
			case 'L':
				if(v_mode){	printf("\n%c %x,%d ",identifier,address,size);}
				accessData(Mycache,lines,address);
				break;
			case 'M':
				if(v_mode){	printf("\n%c %x,%d ",identifier,address,size);}
				accessData(Mycache,lines,address);
				accessData(Mycache,lines,address);
				break;
			case 'S':
				if(v_mode){	printf("\n%c %x,%d ",identifier,address,size);}
				accessData(Mycache,lines,address);
				break;
            		default:
                		break;
		}
	}
	fclose(fp);
}

void printSummary(int hit_count,int miss_count,int eviction_count){
	printf("hits:%d misses:%d evictions:%d\n",hit_count,miss_count,eviction_count);
}

int main(int argc,char* argv[])
{
    int size,S,i;
    char opt; 
    cache_line** Mycache;
    while(1)
    {
	   opt=getopt(argc, (char *const *)argv, "s:E:b:t:vh");
	   if(opt==-1)
		break;
	   switch(opt){
		   case 'h':help();
		   	    break;
		   case 'v':v_mode=1;
			    break;
		   case 's':s=atoi(optarg);
		   	    break;
		   case 'E':lines=atoi(optarg);
		   	    break;
		   case 'b':offset=atoi(optarg);
		   	    break;
		   case 't':strcpy(filename,optarg);
			    break;
		   default:
		   	    printf("./csim: invalid option -- %s\n",optarg);
			    help();
		   	    exit(-1);
	   }
    }
    S=1<<s;
    printf("%d\n",S);
    printf("%d\n",sizeof(cache_line));
    Mycache=(cache_line**)malloc(sizeof(cache_line)*S);
    for(i=0;i<S;i++){	
		Mycache[i]=(cache_line*)malloc(sizeof(cache_line)*lines);
	}
    initData(Mycache);
    simulator(Mycache,filename,S,lines);
    for(i=0;i<S;i++){
		free(Mycache[i]);
		Mycache[i]=NULL;
	}
    free(Mycache);
    Mycache=NULL;
    printSummary(hit_count,miss_count,eviction_count);
    return 0;
}
```

<h2>PART B</h2>

​		优化函数转置矩阵，使得cache miss尽可能少。

