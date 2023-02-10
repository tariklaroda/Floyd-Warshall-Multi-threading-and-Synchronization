/*
   Tarik LaRoda  A00440772
   Project 2 
   Floyd Warshall Multithreading
*/

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <sys/time.h>
#include <time.h> 

#define INF 999

//make double pointer for graph and dist 2D arrays
int **graph;
int **dist;

//the lock used in multithreading
pthread_mutex_t distMutex; 

//struct to hold args
struct args {
    int n; //number of nodes
    int i; // ith iteration
    int k; // kth iteration
};

/*
    This function checks to see if a number n is positive;. Returns 1 if true.
*/
int checkNum(int n){
    return n > 0;
}



/*
    This function checks to make sure that the matrix is not too large
*/

int checkMatrix(int numOfNodes, int numOfEdges){
    if(checkNum(numOfNodes)&&checkNum(numOfEdges)){
        int max = 0;
        for(int i = 0; i<numOfNodes;i++){
            max+=i;
        }
        return (max > numOfEdges);
    } else {
        return 0;
    }
}

/*
    This function displays the Matrix
*/
void displayMatrix(struct args *argument, int a){
    
   // initialize the number of nodes
   int n = argument[0].n;
  if (a == 0) {
      printf("\nShortest Path: multithreading\n");
   
   for (int i = 0; i < n; i++){
       
       for (int j = 0; j < n; j++){
           
           if(dist[i][j] == INF){
               printf("INF ");
           } else {
               printf("%d ", dist[i][j]);
           }
       }
       printf("\n");
   }
      
  } else {
       printf("\nShortest Path: single-threading\n");
   
   for (int i = 0; i < n; i++){
       
       for (int j = 0; j < n; j++){
           
           if(graph[i][j] == INF){
               printf("INF ");
           } else {
               printf("%d ", graph[i][j]);
           }
       }
       printf("\n");
   }
  }
   
   
}



 /*
    Worker function for multithreading implementation
    @args struct args
 */
void *worker (void *args){
    
    // get n, i , k from args
    struct args *argument = args;
    int n = argument->n;
    int i = argument->i;
    int k = argument->k;
    
    for (int j = 0; j < n; j++){
        
        //aquire read lock
        pthread_mutex_lock(&distMutex);
        
        if ((dist[i][k] + dist[k][j]) < dist[i][j]){
            
            //release read lock
            pthread_mutex_unlock(&distMutex);
            
            //aquire write lock
            pthread_mutex_lock(&distMutex);
            dist[i][j] = dist[i][k] + dist[k][j];
            
            //release write lock
            pthread_mutex_unlock(&distMutex);
        } else {
            //release read 
            pthread_mutex_unlock(&distMutex);
        }
    }
    pthread_exit(NULL);
  
}
/*
    This worker function for single-threading implementation
    @args struct args
*/
void *workerSingleThreaded(void *args){
    
    struct args *argument=(struct args *)args;
    int n=argument[0].n;

    for(int k=0;k<n;k++)
    {
        for(int i=0;i<n;i++)
        {
            for(int j=0;j<n;j++)
            {
                if((graph[i][k]+graph[k][j])<graph[i][j])
                    graph[i][j]=graph[i][k]+graph[k][j];
            }
        }
    }
    pthread_exit(NULL);
}

/*
    Function to calculate the shortest path using MultiThreading
    @argument struct args
*/
void findShortestPath (struct args *argument){
    
    double totalTime = 0.0;
    double totalTimeMulti = 0.0;
    //initialize mutex
    
    pthread_mutex_init(&distMutex, NULL);
    
    
    
    int numOfNodes = argument[0].n;
    printf("----Multi-threading----\n");
    //Allocate memory for threads
    
    pthread_t *th;
    th = (pthread_t *)malloc(numOfNodes * sizeof(pthread_t *));
    
    for (int k = 0; k < numOfNodes; k++){
        totalTime = 0.0;
        //create threads
        for(int i = 0; i < numOfNodes; i++){
            //initialize k and i
            //each thread should be aware of the current kth & ith iteration
        
            argument[i].k = k;
            argument[i].i = i;
            pthread_create(th+i, NULL, worker, (void *)&(argument[i]));
        }
        
       
        //begin clock
        clock_t begin = clock();
        
        //join threads
        for (int i = 0; i < numOfNodes; i++){
            pthread_join(*(th+i),NULL);
        }
        //stop clock
        clock_t end = clock();
        totalTime = (double)(end - begin) / CLOCKS_PER_SEC;
        
        // add to total time of all threads
        totalTimeMulti += totalTime;
        
        printf("Thread %d finished in %f seconds \n", k+1, totalTime);
        
        
      
        
    }
    printf("Finished multithreading in %f s\n\n", totalTimeMulti);
    
    

    //single-threading
    printf("----Single-threading----\n");
    totalTime = 0.0;
    pthread_t tid[1];

    pthread_create(&tid[0],NULL,workerSingleThreaded,(void *)&(argument[0]));
    clock_t begin = clock();
    pthread_join(tid[0],NULL);
    clock_t end = clock();
    totalTime = (double)(end - begin) / CLOCKS_PER_SEC;
    
    printf("Finished single-threading in %f s \n", totalTime);

    
} 
/*
    Main function
*/
int main()
{
    // create struct 
    struct args *argument;
    
    
     //declare numOfNodes and numOfEdges
    int numOfNodes = 0;
    int numOfEdges = 0;
    
    do {
         //input number of nodes and number of edges
    printf("Enter N and M:\n");
    scanf("%d %d", &numOfNodes, &numOfEdges);
    // printf("N = %d, M = %d\n", numOfNodes, numOfEdges);
    
        if (!checkMatrix(numOfNodes,numOfEdges)){
            printf("Error: There is a negative number or the number of edge exceeds the maximum number\n");
        }
    } while (!checkMatrix(numOfNodes, numOfEdges));
   
    
    //Allocate memory for arguments, graph and dist
    argument = (struct args *)malloc(numOfNodes * sizeof(struct args));
    graph = (int **)malloc(numOfNodes * sizeof(int *));
    dist = (int**)malloc(numOfNodes * sizeof(int *));
    
    
    for (int i = 0; i < numOfNodes; i++){
        
        //initialize n
        argument[i].n = numOfNodes;
        
        //allocate memory for 1d array
        graph[i] = (int *)malloc(numOfNodes * sizeof(int *));
        dist[i] = (int *)malloc(numOfNodes * sizeof(int *));
        
        for (int j = 0; j < numOfNodes; j++){
            
            if (i == j){
                //if the indexes are the same, the distance is 0.
                graph[i][j] = 0;
                dist[i][j] = 0;
                
            } else {
                //fill remaining 2D elements with INF
                graph[i][j]= INF;
                dist[i][j] = INF;
                
            }
        }
    }
    
    //get weight of Edges
    printf("Enter weight of edges:\n");
    
    for (int i = 0; i < numOfEdges; i++){
        
        int u; // node 1
        int v; // node 2
        int w; // weight between node 1 and node 2D
        
        //get inputs 
        scanf("%d %d %d", &u, &v, &w);
        
        //check if numbers are negative
       if (!(checkNum(u)&&checkNum(v)&&checkNum(w))){
           printf("This line contains illegal negative(s)\n");
           printf("Input again\n");
          i--;
       } else if(u > numOfNodes || v > numOfNodes){
           printf("Error: This u and v must be less than the total amount of nodes.\n");
           printf("Input again\n");
           i--;
       } else {
          graph[u - 1][v - 1] = w;
          graph[v - 1][u - 1] = w;
           dist[u - 1][v - 1] = w;
           dist[v - 1][u - 1] = w;
            
       }
       
       
    }
/*
// Display Dist Matrix

printf("\nThe Dist Matrix:\n");
    for(int i = 0; i < numOfNodes; i++){
    for(int j = 0; j < numOfNodes; j++){
        
        if (dist[i][j] == INF){
            printf("INF ");
        } else {
           printf("%d ", dist[i][j]); 
        }
    }
    printf("\n");
    
}
*/
    
    // find shortest path through FLoyd-Warshall Algorithm
    findShortestPath(argument);
    
    //displayMatrix multithreaded dist
    displayMatrix(argument, 0);
 
  //displayMatrix - singlethreaded graph
    displayMatrix(argument, 1);
 

    return 0;
}