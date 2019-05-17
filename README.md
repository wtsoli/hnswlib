HNSW.Java
=========


Work in progress pure Java implementation of the [the Hierarchical Navigable Small World graphs](https://arxiv.org/abs/1603.09320) algorithm for doing approximate nearest neighbour search.

The index is thread safe and supports adding items to the index incrementally. 

It's flexible interface makes it easy to apply it to use it with any type of data and distance metric  


Code example:


    Index<String, float[], Word, Float> index =
        new HnswIndex.Builder<>(DistanceFunctions::cosineDistance, words.size())
            .build();

    index.addAll(words);
    
    Word item = index.get("king");
    
    List<SearchResult<Word, Float>> nearest = index.findNearest(item.vector, 10);
    
    for (SearchResult<Word, Float> result : nearest) {
        System.out.println(result.getItem().getId() + " " + result.getDistance());
    }


Frequently asked questions
--------------------------

- Will [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions be used ?

  It depends on the jvm implementation because until project [JEP-338](https://openjdk.java.net/jeps/338) is completed you 
  cannot use SIMD explicitly from java. With the oracle / open jdk you can pass the following options to view the assembly 
  code generated by the JIT 

      -XX:+UseSuperWord -XX:+UnlockDiagnosticVMOptions -XX:CompileCommand=print,*DistanceFunctions.cosineDistance

  For more information consult [Vectorization in HotSpot JVM](https://cr.openjdk.java.net/~vlivanov/talks/2017_Vectorization_in_HotSpot_JVM.pdf)


- How much memory is used?

  Rather than providing you with a complicated formula that takes many variables into account. I suggest 
  using using [Java Agent for Memory Measurements](https://github.com/jbellis/jamm) to measure actual object
  memory use including JVM overhead. Here's an example of how to do this :
  
        import org.github.jamm.MemoryMeter;
        import org.github.jelmerk.knn.DistanceFunctions;
        import org.github.jelmerk.knn.Index;
        
        import java.util.List;
        
        public class MemoryMeasurement {
       
            public static void main(String[] args) throws Exception {
                List<MyItem> allElements = loadItemsToIndex();
        
                int increment = 100_000;
                long lastSeenMemory = -1L;
        
                for (int i = increment; i <= allElements.size(); i += increment) {
                    List<MyItem> items = allElements.subList(0, i);
        
                    long memoryUsed = createIndexAndMeasureMemory(items);
        
                    if (lastSeenMemory == -1) {
                        System.out.printf("Memory used for index of size %d is %d bytes%n", i, memoryUsed);
                    } else {
                        System.out.printf("Memory used for index of size %d is %d bytes, delta with last generated index : %d bytes%n", i, memoryUsed, memoryUsed - lastSeenMemory);
                    }
                    
                    lastSeenMemory = memoryUsed;
                    createIndexAndMeaureMemory(items);
                }
            }
        
            private static long createIndexAndMeasureMemory(List<MyItem> items) throws InterruptedException {
                MemoryMeter meter = new MemoryMeter();
        
                Index<Integer, float[], MyItem, Float> index =
                        new HnswIndex.Builder<>(DistanceFunctions::cosineDistance, items.size())
                                .setM(16)
                                .build();
                index.addAll(items);
                
                return meter.measureDeep(index);
            }
 
   Run the above code with -javaagent:/path/to/jamm-0.3.0.jar 
   
   The output of this program will show approximately how much memory adding an additional 100.000 elements to this index will take up
   Since the amount of memory used scales roughly linearly with the amount of elements you should be able to work out your memory requirements 
   