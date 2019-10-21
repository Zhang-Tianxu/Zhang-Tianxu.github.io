---
title: Producer-consumer problem solution in C
date: 2019-10-21 19:43:20
tags:
  - Producer-consumer problem
  - concurrency
  - multithreading
  - multiprocessing
  - C
categories:
  - Coding
  - C/C++
---

# Producer-consumer problem multiprocessing solution in C with details

## Problem

[**Producer-consumer problem**]([https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem](https://en.wikipedia.org/wiki/Producer–consumer_problem)) is a classical problem in concurrency programming, I want to solve it by multiprocessing method. Multiprocessing method need more knowledges than multithreading method, because different process need to communicate with each other. 

Process can communicate with each other by **IPC** ([*Inter-Process Communication*](https://en.wikipedia.org/wiki/Inter-process_communication)), I chose a type of IPC , **share memory**, to implement.

<!--more-->

## Solutions

The key to solve the producer-consumer problem is synchronization, there is another point that seems easy to solve, that is comsumer comsume producer's products. In multiprocessing case, comsumer process have to communicate with producer process to know where product is. So, 2 keys to solve the problem:

* Synchronization 
* Communication

In C language, header file [*<sys/sem.h>*](https://pubs.opengroup.org/onlinepubs/7908799/xsh/syssem.h.html) helps us solve the first one, header file [*<sys/shm.h>*](https://pubs.opengroup.org/onlinepubs/009695399/basedefs/sys/shm.h.html) help us solve the second one.

## Implementation

I defined two types in `my_type.h`:

1. `product`: simulate produced products.
2. `shm_manage`: used to shared memory, indicate used memory range in shared memory.

```c
// my_type.h
#ifndef _MY_TYPE_H
#define _MY_TYPE_H

typedef struct {
    char name[20];
    int id;
} product;

typedef struct {
    int start;
    int end;
} shm_manage;
#endif
```

I wrapped 4 method of semaphore operation:

1. `set_sem_value`: set semaphore `sem_id`'s `sem_num` to `value`
2. `del_sem_value`: delete semaphore `sem_id`
3. `sem_p`: minus 1 from semaphore `sem_id`'s `sem_num`
4. `sem_v`: add 1 to semaphore `sem_id`'s `sem_num`

```c
// my_sem_ops.h
#ifndef _MY_SEM_OPS_H
#define _MY_SEM_OPS_H

#include <sys/sem.h>

static int set_sem_value(int sem_id,int sem_num, int value)
{
    // init semaphore
    union semun sem_union;
    sem_union.val = value;
    if(semctl(sem_id,sem_num,SETVAL,sem_union) < 0)
    {
        perror("init semaphore error:");
        return -1;
    }
    return 0;

};

static void del_sem_value(int sem_id)
{
    // delete semaphore
    union semun sem_union;
    if(semctl(sem_id,0,IPC_RMID,sem_union) < 0)
    {
        perror("delete semaphore error:");
        return;
    }
    return;
};

static int sem_p(int sem_id, int sem_num)
{
    struct sembuf sem_b;
    sem_b.sem_num = sem_num;
    sem_b.sem_op = -1;//P()
    sem_b.sem_flg = SEM_UNDO;
    if(semop(sem_id, &sem_b,1) < 0)
    {
        perror("p() error:");
        return -1;
    }
    return 0;
};

static int sem_v(int sem_id,int sem_num)
{
    struct sembuf sem_b;
    sem_b.sem_num = sem_num;
    sem_b.sem_op = 1;//V()
    sem_b.sem_flg = SEM_UNDO;
    if(semop(sem_id, &sem_b,1) < 0)
    {
        perror("p() error:");
        return -1;
    }
    return 0;
};

#endif
```

In producer:

* ftok() use a file to get a IPC id, I used file '.', that means producers and consumers must run in same directory, or they will have differ IPC id.
* `semopts.buf->sem_otime == 0` to determine whether it's first producer, first producer have to initiate three semaphores.
  * only `union semun semopts;` is not enough, will cause `semctl()`'s error: bad address.
  * Need `struct semid_ds mysemds;semopts.buf = &mysemds;`

```c
// producer.c
#include <sys/sem.h>
#include <sys/shm.h>
#include <unistd.h>
#include <stdio.h>
#include <signal.h>
#include <string.h>

#include "my_sem_ops.h"
#include "my_type.h" 

#define STORAGE_ROOM 10
#define BUFFER_SIZE sizeof(product) * STORAGE_ROOM + sizeof(shm_manage)

int sem_id; // semaphore's id
int shm_id; // share memory's id

union semun semopts;

shm_manage *shm_m; // data structure that used to manage shared memory
product *product_start_addr; // start address in share memory used to store product's info


void sig_handler(int signo) { // handle interrupt
    printf("Caught signal %d\n",signo);
    if (shmctl(shm_id, IPC_RMID, 0) == -1)
    {
        fprintf(stderr, "shmctl(IPC_RMID) failed\n");
        return;
    }
    del_sem_value(sem_id);
    return;
};

// producer to produce products
void produce(int product_number)
{
    product p;
    strncpy(p.name,"beer",sizeof("beer"));
    p.id = product_number;
    
    shm_m->end = (shm_m->end + 1) % STORAGE_ROOM;
    
    strncpy((product_start_addr + shm_m->end)->name,p.name,strlen(p.name));
    printf("producing: product's id is %d, product's name is %s\n",(product_start_addr + shm_m->end)->id ,(product_start_addr + shm_m->end)->name);
};


int main()
{
    // set signal handler
    struct sigaction sigIntHandler;
    sigIntHandler.sa_handler = sig_handler;
    sigemptyset(&sigIntHandler.sa_mask);
    sigIntHandler.sa_flags = 0;

    // generate key
    key_t key;
    key = ftok(".",0x01);
    if(key < 0)
    {
        perror("ftok error:");
        return -1;
    }
    printf("key = %d\n",key);

    // generate shm_id
    shm_id = shmget(key,BUFFER_SIZE,IPC_CREAT | 0600);
    if(shm_id < 0)
    {
        perror("shmget error:");
        return -1;
    }
    printf("shm_id = %d\n",shm_id);


    // generate sem_id
    // 3 semaphores
    // 0 - Remaining room
    // 1 - produced product number
    // 2 - mutex lock
    sem_id = semget(key,3,IPC_CREAT | 0600); // get semaphore
    if(sem_id < 0)
    {
        perror("semget error:");
        return -1;
    }
    printf("sem_id = %d\n",sem_id);

    struct semid_ds mysemds;
    semopts.buf = &mysemds;

    if(semctl(sem_id,1,IPC_STAT,semopts) < 0)
    {
        perror("semctl error!");
        return -1;
    }

    if(semopts.buf->sem_otime == 0)
    {
        printf("I am the first producer\n");
        if( set_sem_value(sem_id,0,STORAGE_ROOM) < 0) // set semaphore sem_id's first semaphore's initia value to STORAGE
        {
            printf("Failed to initialize semaphore\n");
            return -1;
        }
        if( set_sem_value(sem_id,1,0) < 0) //sem_id's second semaphore
        {
            printf("Failed to initialize semaphore\n");
            return -1;
        }
        if( set_sem_value(sem_id,2,1) < 0)
        {
            printf("Failed to initialize semaphore\n");
            return -1;
        }
    }

    // map at 
    shm_m = (shm_manage*)shmat(shm_id,NULL,0);
    product_start_addr = (product *)(shm_m + 1);


    // producing
    int product_number = 0;
    while(true)
    {
        if(sem_p(sem_id,0) < 0)
        {
            printf("sem_p error!\n");
            return -1;
        }
        if(sem_p(sem_id,2) < 0)
        {
            printf("sem_p error!\n");
            return -1;
        }
        // critical area


        produce(product_number++);
        sleep(1);


        // -------------

        if(sem_v(sem_id,2) < 0)
        {
            printf("sem_v error!\n");
            return -1;
        }
        if(sem_v(sem_id,1) < 0)
        {
            printf("sem_v error!\n");
            return -1;
        }
        sigaction(SIGINT,&sigIntHandler,NULL);
    }


    shmdt(shm_m);
    del_sem_value(sem_id);


}
```

```c
// consumer.c
#include <sys/sem.h>
#include <sys/shm.h>
#include <unistd.h>
#include <stdio.h>
#include <signal.h>
#include <string.h>

#include "my_sem_ops.h"
#include "my_type.h" 

#define STORAGE_ROOM 10
#define BUFFER_SIZE sizeof(product) * STORAGE_ROOM + sizeof(shm_manage)

int sem_id; // semaphore's id
int shm_id; // share memory's id

shm_manage *shm_m; // data structure that used to manage shared memory
product *product_start_addr; // start address in share memory used to store product's info


void sig_handler(int signo) { // handle interrupt
    printf("Caught signal %d\n",signo);
    // shmdt(p_map);
    //del_sem_value(sem_id);
    return;
};

void consume()
{
    shm_m->start = (shm_m->start + 1) % STORAGE_ROOM;
    
    printf("consuming: product's is %d, product's name is %s\n",(product_start_addr + shm_m->start)->id,(product_start_addr + shm_m->start)->name);
};


int main()
{
    // set signal handler
    struct sigaction sigIntHandler;
    sigIntHandler.sa_handler = sig_handler;
    sigemptyset(&sigIntHandler.sa_mask);
    sigIntHandler.sa_flags = 0;

    // generate key
    key_t key;
    key = ftok(".",0x01);
    if(key < 0)
    {
        perror("ftok error:");
        return -1;
    }
    printf("key = %d\n",key);

    // generate shm_id
    shm_id = shmget(key,BUFFER_SIZE,0600);
    if(shm_id < 0)
    {
        perror("shmget error:");
        return -1;
    }
    printf("shm_id = %d\n",shm_id);


    // generate sem_id
    // 3 semaphores
    // 0 - Remaining room
    // 1 - produced product number
    // 2 - mutex lock
    sem_id = semget(key,3,0600); // get semaphore
    if(sem_id < 0)
    {
        perror("semget error:");
        return -1;
    }
    printf("sem_id = %d\n",sem_id);

    // map at 
    shm_m = (shm_manage*)shmat(shm_id,NULL,0);
    product_start_addr = (product *)(shm_m + 1);


    // producing
    int product_number = 0;
    while(true)
    {
        if(sem_p(sem_id,1) < 0)
        {
            printf("sem_p error!\n");
            return -1;
        }
        if(sem_p(sem_id,2) < 0)
        {
            printf("sem_p error!\n");
            return -1;
        }
        // critical area
        
        
        //produce(product_number++);
        consume();
        sleep(1);
        

        // -------------

        if(sem_v(sem_id,2) < 0)
        {
            printf("sem_v error!\n");
            return -1;
        }
        if(sem_v(sem_id,0) < 0)
        {
            printf("sem_v error!\n");
            return -1;
        }
        sigaction(SIGINT,&sigIntHandler,NULL);
    }


    shmdt(shm_m);
    del_sem_value(sem_id);


}
```

