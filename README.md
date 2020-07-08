# Ranker-Worker Architecture

![Ranker-worker architecture diagram](/images/architecture.PNG)

Usually when designing a solution, we thing about decoupling the parts, so all components can scale, improving resilience and performance.

But, when you  have a large table, how can you process this set of data in a way that you can run the parallel process that are ready to work? On single select can put your resources down.

## Ranker

As the name said, this component will take advantage of Azure SQL Server [ranking functions](https://docs.microsoft.com/en-us/sql/t-sql/functions/ntile-transact-sql?view=sql-server-ver15) . In my case, I used the NTILE command which do the following:

> Distributes the rows in an ordered partition into a specified number of groups. The groups are numbered, starting at one. For each row, NTILE returns the number of the group to which the row belongs.

In other others, NTILE will take your data, and separate it according the value you add. Example:

###  Before NTILE

| ID        | Name           | Rate  |
| ------------- |:-------------:| -----:|
| 1      | Ernesto  | $30     |
| 2      | Fabio      |   $30 |
| 3      | Joao      |    $30 |
| 4      | Mario      |    $30 |
| 5      | Mike     |    $30 |


###  After NTILE

Applying NTILE(3) - Check the sintax and examples [here](https://docs.microsoft.com/en-us/sql/t-sql/functions/ntile-transact-sql?view=sql-server-ver15)

| ID        | Name           | Rate  | Rank
| ------------- |:-------------:| -----:| -----:| 
| 1      | Ernesto  | $30     | 1
| 2      | Fabio      |   $30 | 1
| 3      | Joao      |    $30 | 2
| 4      | Mario      |    $30 |2
| 5      | Mike     |    $30 | 3


If you observe, NTILE took the data according with the informed value and ranked the data, equally. In this way, if you want to have 100 workers taking slices of this data, we can add NTILE (100)

One example showing the sintax of NTILE. 

```
SELECT [CustomerID]      
      ,[Title]
      ,[FirstName]
      ,[MiddleName]
      ,[LastName]   
	  , NTILE(2) OVER(ORDER BY [LastName] DESC) AS Rank  
FROM [SalesLT].[Customer]
```

 Why is it important now? The large amount of information now can have a columns where the worked can search for and retrieve only a subset of information.

 ### Add the job ticket to worker

 Here is the trick. When the Ranker finish the ranking job, it puts in a Queue messages to the workers. So, if you choose NTILE(20) you need to add 20 messages which the body from each message will contains a numer. Like 1,2,3,4... 20. This will instruct the workers in what to do.


## Worker

The component needs to execute whatever it is reponsable for. Usually on cloud solution, you start with only one instance of this worker. It´s listening for the messages, so the information flow should be like this.

1. Worker receives the message and check the number inside: 1
2. It selects all data from large table where Rank=1
3. Some async parallel process start now
4. It gets other message, but now, the body contains 2.
5. Process repeat again

If the resources get compromissed, the auto scale from cloud providers will happend now. Meanwhile the 1st worker is busy dealing with rank 1 and rank 2, the third one starts, look for the Queue and start to process Rank 3.


## Tips


1.  A good way to find the NTILE nuber is to define how many records do your work can process. Supposing that it can handles 10.000 lines and you have 1M records. You can calculate : 1.000.000 / 10.000 = 100. Your NTILE will be 100.

2. To achieve the scalability, you can´t wait the worker to finish an amount of pages to get a next message. It will be blocked processing a page instead of getting pages and using the server resources for that.



