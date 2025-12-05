# with-go-defer

Go-style defer functionality for JavaScript and TypeScript. Declare cleanups right next to resource acquisition, and they'll execute automatically in the correct order when your function exits.

## Quick Start

```bash
npm install with-go-defer
```

```javascript
import { withDefer } from 'with-go-defer';

await withDefer(async (defer) => {
  const file = openFile('data.txt');
  defer(() => file.close());  // Cleanup declared right here
  
  const db = await connectDatabase();
  defer(async () => await db.close());
  
  return await processData(file.read());
  // Both resources cleaned up automatically in reverse order
});
```

## Why defer?

Managing resource cleanup becomes error-prone when you have multiple exit paths:

```javascript
function processData() {
  const file = openFile('data.txt');
  const conn = connectDB();
  
  try {
    const data = file.read();
    if (!isValid(data)) return null;
    
    const result = transform(data);
    if (result.error) throw result.error;
    
    return result;
  } finally {
    conn.close();  // Cleanup is far from acquisition
    file.close();  // Hard to see what resources are managed
  }
}
```

With defer, cleanup is paired with acquisition:

```javascript
await withDefer((defer) => {
  const file = openFile('data.txt');
  defer(() => file.close());  // Obvious pairing
  
  const conn = connectDB();
  defer(() => conn.close());  // Clear ownership
  
  const data = file.read();
  if (!isValid(data)) return null;
  
  const result = transform(data);
  if (result.error) throw result.error;
  
  return result;
  // Cleanup: conn.close(), then file.close() - automatic and correct
});
```

## Key Benefits

**Cleanup stays with acquisition.** When you acquire a resource, you immediately declare how to clean it up. No scrolling to a distant finally block to understand resource management.

**Guaranteed execution.** Cleanup runs on every exit path - normal return, early return, or thrown error. You declare it once and it always happens.

**Correct order automatically.** Resources clean up in reverse order (LIFO). If resource B depends on resource A, and you acquire A then B, defer ensures B cleans up before A. This happens automatically without manual ordering.

**Refactoring safety.** Add early returns or change control flow without updating cleanup code. The cleanup you declared keeps working.

## API Reference

### withDefer(fn)

Executes `fn` with a defer context. Deferred functions execute in LIFO order when `fn` exits.

```typescript
function withDefer<T>(
  fn: (defer: (callback: () => void | Promise<void>) => void) => T | Promise<T>
): Promise<T>
```

**Parameters:**
- `fn` - Function receiving a `defer` callback for registering cleanup functions

**Returns:** Promise resolving to the return value of `fn`

**Behavior:**
- Deferred functions execute in reverse order (last deferred runs first)
- All deferred functions execute even if the main function throws
- All deferred functions execute even if some deferred functions throw
- If errors occur, they're collected into an `AggregateError`

**Example:**

```javascript
await withDefer(async (defer) => {
  defer(() => console.log('Third'));
  defer(() => console.log('Second'));
  defer(() => console.log('First'));
  console.log('Main');
});
// Output: Main, First, Second, Third
```

### Error Handling

If the main function or any deferred function throws, all deferred functions still execute. Errors are collected:

```javascript
try {
  await withDefer((defer) => {
    defer(() => { throw new Error('Cleanup 1 failed'); });
    defer(() => console.log('This still runs'));
    defer(() => { throw new Error('Cleanup 2 failed'); });
    
    throw new Error('Main failed');
  });
} catch (error) {
  console.log(error instanceof AggregateError); // true
  console.log(error.errors.length); // 3
  // Errors: [Main failed, Cleanup 2 failed, Cleanup 1 failed]
}
```

## Comparison with Go

This library implements Go's `defer` pattern for JavaScript:

**Go's defer behavior:**
- ✅ LIFO execution order (last deferred runs first)
- ✅ Executes on all exit paths (return, throw)
- ✅ Return values from deferred functions are ignored
- ✅ All deferred functions execute even if some fail

## License

MIT
