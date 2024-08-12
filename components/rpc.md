# RPC

`RPC` (Remote Procedure Call), which is used to facilitate communication between a web page and an iframe (or other browser windows) using the `postMessage` API. This allows for sending and receiving messages and invoking methods across different execution contexts. Here's a detailed breakdown of how the `RPC` class works:

#### Class Properties

1. **target**: The target window (or iframe) to which messages will be sent.
2. **rpcId**: A counter used to generate unique IDs for each message sent. This helps track responses to asynchronous calls.
3. **callbacks**: An object that stores callback functions associated with specific message IDs, allowing for handling asynchronous responses.
4. **readyQueue**: An array of functions that should be executed once the RPC is ready (i.e., when the iframe signals that it has loaded).
5. **readyStatus**: A boolean flag indicating whether the RPC is ready for communication.
6. **methods**: An object that holds the methods that can be invoked remotely by the target.

{% code overflow="wrap" %}
```typescript
import {bind, slice} from "../helpers";

declare type ReadyFcn = () => any

export class RPC {

    target: any;
    rpcId: number;
    callbacks: any;
    readyQueue: ReadyFcn[];
    readyStatus: boolean;
    methods: { [id: string]: (args?: any) => any };

    constructor(target: any) {

        this.rpcId = 0;
        this.target = target;
        this.callbacks = {};
        this.readyQueue = [];
        this.readyStatus = false;
        this.methods = {};
        bind(window, "message", (e: MessageEvent) => {
            this.message(e);
        });
    }

    startSession() {
        this.sendMessage("frameReady", []);
        return this.frameReady()
    }

    private frameReady() {
        let callbacks, cb, i, len;
        this.readyStatus = true;
        callbacks = this.readyQueue.slice(0);
        for (i = 0, len = callbacks.length; i < len; i++) {
            cb = callbacks[i];
            cb()
        }
        return false
    }

    invoke(...a: any[]) {
        let method: string;
        let args: any[];
        method = arguments[0], args = 2 <= arguments.length ? slice.call(arguments, 1) : [];
        return this.ready(() => {
            return this.sendMessage(method, args);
        });
    }

    processMessage(data: any) {
        let base, name, result;
        try {
            data = JSON.parse(data)
        } catch (error) {
            return
        }
        if (["frameReady", "frameCallback", "isAlive"].indexOf(data.method) !== -1) {
            result = null;

            switch (data.method) {
                case "frameReady":
                    this.frameReady.apply(this, data.args);
                    break;
                case "frameCallback":
                    this.frameCallback.apply(this, data.args);
                    break;
                case "isAlive":
                    this.isAlive.apply(this, data.args);
                    break;
                default:
                // TODO: log method not found
            }
        } else {
            result = typeof(base = this.methods)[name = data.method] === "function" ? base[name].apply(base, data.args) : void 0
        }
        if (data.method !== "frameCallback") {
            return this.invoke("frameCallback", data.id, result)
        }
    }

    ready(fn: ReadyFcn) {
        if (this.readyStatus) {
            return fn()
        } else {
            return this.readyQueue.push(fn)
        }
    }

    message(e: MessageEvent) {
        let shouldProcess;
        shouldProcess = false;
        try {
            shouldProcess = e.source === this.target
        } catch (error) {
        }
        if (shouldProcess) {
            return this.processMessage(e.data)
        }
    }

    isAlive(): boolean {
        return true;
    }

    frameCallback(id: string, result: any): boolean {
        let base;
        if (typeof(base = this.callbacks)[id] === "function") {
            base[id](result)
        }
        delete this.callbacks[id];
        return true
    }

    sendMessage(method: string, args: any[]) {
        let err, id, message, ref;
        if (args == null) {
            args = []
        }
        id = ++this.rpcId;
        if (typeof args[args.length - 1] === "function") {
            this.callbacks[id] = args.pop()
        }
        message = JSON.stringify({method: method, args: args, id: id});
        if (((ref = this.target) != null ? ref.postMessage : void 0) == null) {
            err = new Error("Unable to communicate with Lightbox. Please contact support@monri.com if the problem persists.");
            if (this.methods.rpcError != null) {
                this.methods.rpcError(err)
            } else {
                throw err
            }
            return
        }
        this.target.postMessage(message, "*");
        return message;
    }
}


```
{% endcode %}

#### Constructor

The constructor initializes the RPC object with a target and sets up an event listener for `message` events on the window object. The `message` handler is bound to the `message` method of the RPC instance.

#### Methods

1. **startSession**: Initiates the communication session by sending a "frameReady" message to the target and then calling `frameReady()` to handle the callbacks in the ready queue.
2. **frameReady**: Marks the RPC as ready, executes all functions in the ready queue, and then clears the queue.
3. **invoke**: Invokes a method on the target, passing along any arguments. It ensures the RPC is ready before sending the message.
4. **processMessage**: Processes incoming messages by parsing the data and calling the appropriate method or handling the callback response.
5. **ready**: Adds a function to the ready queue if the RPC is not yet ready. If the RPC is ready, it immediately executes the function.
6. **message**: Handles `message` events from the window, checks if the message is from the target, and processes the message.
7. **isAlive**: A placeholder method that simply returns `true`, possibly used as a heartbeat or sanity check.
8. **frameCallback**: Handles callback responses from the target, executing the associated callback function stored in `callbacks` and then removing it.
9. **sendMessage**: Sends a message to the target with a specified method and arguments. It generates a unique ID for the message, optionally stores a callback function, and uses `postMessage` to send the message. If communication with the target fails, it throws an error or invokes a custom error handler if defined.

#### Summary

The `RPC` class provides a structured way to handle inter-window communication, allowing for method invocation, asynchronous callbacks, and ensuring that messages are processed in order. It handles message parsing, error management, and the synchronization of method calls across different execution contexts, making it a robust solution for integrating iframe components with their parent pages.
