# Why setTimeout get to escape from React state batched updates

Let's say we have this piece of code
```jsx
const [value, setValue] = useState()

const onClick = () => {
  setValue('new value')
}
```

## When `setState` is called
1. React constructs the update object with the new value
    
    ```jsx
      const update: Update<S, A> = {
        lane,
        revertLane: NoLane,
        gesture: null,
        action,
        hasEagerState: false,
        eagerState: null,
        next: (null: any),
      };

      // ReactFiberHooks.js:3552-3560
    ```
2. Put it into an update queue called `concurrentQueues` which is used to store state updates
    
    ```jsx
    enqueueConcurrentHookUpdate(fiber, queue, update, lane);
    
    concurrentQueues[concurrentQueuesIndex++] = fiber;
    concurrentQueues[concurrentQueuesIndex++] = queue;
    concurrentQueues[concurrentQueuesIndex++] = update;
    concurrentQueues[concurrentQueuesIndex++] = lane;

    // ReactFiberHooks.js:3565
    ```
    
3. Schedule a re-render on the fiber (marking it as dirty? TBC)
    
    ```jsx
    scheduleUpdateOnFiber(root, fiber, lane);

    // ReactFiberHooks.js:3568
    ```

At this point, the state is not updated yet! 

The state isn't updated immediately when `setState` is called. The update is queued and processed in the next render phase. If you `console.log` the state right after the `setState`, we won't get the latest value it is supposed to be.
```jsx
const onClick = () => {
  setValue('new value')
  console.log(value) <-- get undefined
}
```

That’s why we keep saying `setState` is async.



## When re-renders happens, aka Render phase
1. In the `useState` hook, React will first get the correct hook in the particular Fiber based on its position in the sequence
    
    ```jsx
    const hook = updateWorkInProgressHook();

    // ReactFiberHooks.js:1296
    ```
    
2. Retrieve the pending updates from the hook
    
    ```jsx
    const pendingQueue = queue.pending;
    ```
    
3. Process all the updates in the queue (which is a circular linked list) to calculate the final state value, regardless of how many updates were queued. This is where batching happens.
    
    ```jsx
    do {
      ...
      update = update.next;
    } while (update !== null && update !== first);
    ```
    
4. Return the final calculated state
    
    ```jsx
    return [hook.memoizedState, dispatch];
    ```
    
5. This is the return value from `useState` hook
    
    ```jsx
    const [value, setValue] = useState()
    ```

This is how batching works. It doesn't matter if we call multiple `setState`, eventually all the updates will be processed in one render and calculated to the final value. 

This mechanism allows React to batch multiple state updates together and process them in a single render cycle, which is more efficient than re-rendering for each individual update.
