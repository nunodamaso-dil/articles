# On Speeding up SynciT

*SynciT is an application data migration tool for OutSystems environments*  

Since the first conversations I had with the SynciT Team, parallelism was a hot topic.  
How fast would it be?  Could we come up with a stable model to predict the migration order? Did OutSystems provide enough parallel mechanisms to pull it off?  
Binaries were also a pain during long migrations, taking a long time to transfer and not really moving towards the goal of having referential data properly migrated.
Then there was also the heart of SynciT engine - updating and matching all the migrated auto number foreign keys - often the culprit of timeouts and slow migrations.  
All these topics needed to be addressed to speed up SynciT.

-----

## Parallelism

### Directed Graphs

The first thing needed was a predictive model of the migration sequence. Database models and their relationships can be represented as directed graphs through nodes(entities) and edges(their relation). Actually, replacing RDBMS engines with Graph ones is a power move as relationships become a true first class citizen - see [neo4j](https://neo4j.com/developer/graph-database/) and [aws](https://aws.amazon.com/nosql/graph/).  
Anyways, we implemented a depth first algorithm to extract the directed graph.  
![Graph](../images/SynciTSpeedingUp/Graph.png)
What this achieves in practical terms is that on one hand you have your work simplified to launch your Light BPTs through a known dependency order that preserves referential integrity and on the other hand you are **telling** your execution what to do instead of *asking*, speeding up things.

### Matrices  

Another noticeable aspect is that we can also represent Graphs through matrices (yeah yeah, those ones):
![Matrice](../images/SynciTSpeedingUp/Matrice.jpg)
How cool is that? We end up with a simple 2D array to store our entire relational model. Horizontally we can check which entities an entity points to, while Vertically what entities are pointing to a particular one.  
It is also possible to quickly identify entity silos - see the yellow boxes.  

The downside here is calculating your cyclic references, needed when migrating data, to ensure referential integrity  correctness.  
**It can be achieved by looking for entries on the identity diagonal**. Here we can explicitly see one of them - SalesPerson to itself - Row 4 Column 4.  
But wait, what about the **Person-SalesPerson-Person**? It is not on the diagonal! To find every cyclic in the graph we need to calculate T^n. Whatever is not 0 on the main diagonal is circular.  
Here's another example:  
![Matrice2](../images/SynciTSpeedingUp/Matrice2.png)  
While the cost to *compute* these matrices will be higher than the depth first algorithm, there is a huge gain when **accessing** this structure at run time versus the graph - it's just a 2d array.

### Light BPTs

OutSystems had launched Light BPTs when I joined the SynciT team. While normal BPTs would technically do the job, the speed gains would be marginal, since keeping history records, callbacks, scheduling, process table entries, direct launches, etc.. slows down the entire thing.  
Light BPTs are the perfect mechanism but with a catch. A 3-minute one. You see, fetching data alone for a single batch of records of a single entity can often take as much as 2 minutes or more, based on server latency, columns, attribute sizes, and more, nevermind the remainder of the process of updating fks and inserting the records.
Here's how we setup the parallel migration logic (overly simplified):  
![Parallel](../images/SynciTSpeedingUp/Parallel.png)

We have a dispatcher to launch a new process that migrates data for a certain entity. The migration is a composition of 2 processes - a fetcher, and an executioner.
After that, the processes themselves are responsible to re-launch the migration processes or to call the dispatcher again. A mix between an Orchestra and a Choreography, if a parallelism to micro services patterns can be made here.

-----

## In memory FK updates

SynciT's approach to match auto-numbers between 2 environments is a simple one - insert records, retrieve the ids, store them in a mapping table to map with those from source.
After that, whenever an entity needs to reference a fk, it fetches the new FK Id from said mapping table.

The opportunity to improve here, comes from the matching flow - SynciT used an approach often mentioned in OutSystems forums and blogs, of creating temporary tables to update incoming entity fk records.  
While there's nothing wrong with that, it implies a lot of disk reads and writes because temporary tables are tables after all.  
Furthermore, Oracle's OutSystems user has no permissions to create temporary tables, so that was already a no no for Oracle.  

![InMem](../images/SynciTSpeedingUp/InMem.jpg)

Moving the whole thing to memory involved some caching and taking advantage of OutSystems extensions singleton patterns, but the results show an observable 10x faster fk updates and above. It can actually be a lot more than that, a disk read/write vs memory factors in the 1000x.

-----

## Setting binaries aside

The idea here was pretty simple. Binary files are required to transfer for a completed migration but not required for working data. So we simply moved the binary files to a parallel timer to deal with those, while the main entity migration deals with data and referential integrity.  
This achieves a ready to work environment before the binaries are all downloaded, speeding up the goal. Once the main entity migration finishes, users can start working while the binaries are still downloading.

-----

## How fast then?  

It's complicated.  

Let's discard the in-memory fk update buff from above. In practical terms, if an entity would take 10 seconds to fetch and 10 seconds to migrate, it now takes 10 seconds to fetch and around 1-2 seconds to migrate and that's what we are rolling with. There's more to it, but these gains will already be assumed for what follows.

Let's also discard the binary scenarios. These will be migrated aside as previously mentioned, so these gains are also assumed.

### Example 1

![example1](../images/SynciTSpeedingUp/ex1.png)  

In a sequential fashion, this migration takes 50 seconds.  
In parallel with 5 available LBTs, it takes around 10 seconds.  

### Example 2

![example2](../images/SynciTSpeedingUp/ex2.png)  

Let's switch things up a bit. With the same 5 entities, now one has 900k records and this one takes 50 seconds. The other 4, 10k and they take 1 second.

Sequential - 54 seconds, Parallel - 50 seconds  
Pfff, not much to be said here, it's easy to see why these values are so close - the big one takes most of the time.

### Example 3

![example3](../images/SynciTSpeedingUp/ex3.png)  

Sequential will be equal to Parallel - this shows the relation between sparse and vertical data models and its effect on using parallelism.

### Other considerations  

The are metal considerations as well. How much ram do the fe-servers have? The db server? How many LBPTs are available/feasible to use? Is the Server healthy? All of this needs to be taken into account.  

All and all, for any given random migration we are expecting average speed gains in order of at least 2 to 5x.

-----

## Wrap up  

Here's an age-old technique that I use to map and organize some of this stuff - it involves a pencil, paper and liquids.  
Feel free to reach out if you want to know more about any of these topics!
  
![technique](../images/SynciTSpeedingUp/technique.jpg)
