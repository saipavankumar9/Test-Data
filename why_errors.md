# Understanding Error Propagation in Your Reactive Flow
How errors propagate through reactive chains in Project Reactor, particularly when they cross component boundaries.
Why Error Isn't Being Caught in the BaseExecutor

The code in `BaseExecutor.executeNode()`, error handling for `IllegalArgumentException`, but the error is still propagating to `SerialNode` and then back to `BaseExecutor`. This happens because:

1. The TextSplitter component throws an IllegalArgumentException directly in its code:
```java
@Override
public Mono<Map<String, Object>> executeNode(Node node, ExecutionData executionData) {
    Map<String, Object> inputData = node.getInput();
    throw new IllegalArgumentException("input is empty");  // <-- Direct throw!
    // ...
}
```

2. When you throw an exception directly (not inside a Mono/Flux), it happens during the assembly phase of the reactive pipeline, not during the subscription phase.
3. Error handling in `BaseExecutor` is designed to catch errors during the subscription phase (when the reactive pipeline is running).

We need to make sure exceptions are thrown within the reactive context. Change the `TextSplitter` implementation:
```java
TextSplitter.java
@Override
public Mono<Map<String, Object>> executeNode(Node node, ExecutionData executionData) {
    Map<String, Object> inputData = node.getInput();
    
    // Return a Mono.error instead of throwing directly
    if(inputData == null || inputData.isEmpty()) {
        return Mono.error(new IllegalArgumentException("input is empty"));
    }
    
    String delimiter = (String) inputData.get("delimiter");
    String text = (String) inputData.get("text");
    return Mono.just(Map.of(RESULT_KEY, text.split(delimiter)));
}
```

By returning `Mono.error()` instead of throwing directly, you ensure that:
1. The exception becomes part of the reactive flow
2. It will be emitted during the subscription phase
3. Error handlers in `BaseExecutor` will be able to catch it

### Why This Works
When we use `Mono.error()`, the exception becomes part of the reactive pipeline and will be emitted when the pipeline is subscribed to. 
This allows our error handlers (like `onErrorResume`) to catch and process the exception.

When we throw an exception directly, it happens immediately during method execution, before the reactive pipeline even has a chance to handle it.

This is a common pitfall in reactive programming - distinguishing between exceptions that occur during assembly versus those that occur during subscription.
