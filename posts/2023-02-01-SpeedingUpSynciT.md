# On Speeding up SynciT

*SynciT is an application data migration tool for Outsystems environments*  
Since the first conversations I had with the SynciT Team, the ghost of parallelism was in the air.  
How fast would it be?  Could we come up with a stable model to predict the migration order?  Did Outsystems provide enough parallel mechanisms to pull it off?  
Then there was also the heart of SynciT engine - updating and matching all the migrated auto number foreign keys - often the culprit of timeouts and slow migrations.  
Both these topics were required to address to speed up SynciT.

-----

## Achieving Parallelism

### Directed Graphs

The first thing needed was a predictive model of the migration sequence. Database models and their relationships can be represented as directed graphs (and actually replacing RDBMS engines with Graph ones is a power move) through nodes(entities) and edges(their relation). So we implemented a depth first algorithm to extract the directed graph.  
![Graph](../images/SynciTSpeedingUp/Graph.png)
What this achieves in practical terms is that on one hand you have your work simplified to launch your Light BPTs through a known dependency order and on the other hand you are **telling** your execution what to do instead of *asking*.

An also noticeable aspect is that we can also represent Graphs through matrices (yeah yeah, those ones):
![Graph](../images/SynciTSpeedingUp/Matrice.jpg)
How cool is that? We end up with a simple 2D array to store our entire relational model. Horizontally we can check which entities an entity points to, while Vertically what entities are pointing to a particular one.  
It is also possible to quickly identify entity silos - see the yellow boxes.  
The downside here is calculating your cyclic references, needed when migrating data, to ensure referential integrety correctness.  

It can be achieved by calculating T^n and then looking for entries at the identity diagonal. Here we can see explicity one of them - SalesPerson to itself - Row 4 Column 4.

We can see that the Person-SalesPerson-Person is not on the diagonal. To find every cyclic in the graph we need to calculate T^5. Whatever is not 0 on the main diagonal is circular.  
While the cost to *compute* these matrices will be higher, there is a huge gain when **accessing** this structure at run time versus the graph - its just a 2d array.

### Light BPTs

Outsystems had launched Light BPTs when I joined the SynciT team. While normal BPTs would technically do the job, the speed gains would be marginal, since keeping history records and having callbacks and direct launches slows down the entire thing.
Light BPTs is the perfect mechanism but with a catch. A 3 minutes one. You see, fetching data alone for a single batch of records of a single entity can often take as much as 2minutes or more, nevermind the remainder of the process of updating fks and inserting the records.
Here's how we setup the migration logic:

We have a dispatcher to launch a new process that migrates data for a certain entity. The migration is a composition of 2 processes - a fetcher, and an executioner.
After that, the processes themselves are responsible to re-launch the migration processes or to call the dispatcher again. A mix between an Orchestra and a Choreography(http://) if a parallelism to services can be made.

## In memory FK updates
