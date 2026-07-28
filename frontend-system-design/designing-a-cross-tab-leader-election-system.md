# Designing a Cross-Tab Leader Election System

> **Topic:** Frontend System Design · **Level:** Advanced · **Author:** [@ASANIYAN](https://github.com/ASANIYAN)

## The Problem

A user can open a web app in more than one browser tab at the same time.
Some apps use a live connection to a server, for example a WebSocket.
If each tab opens its own live connection, the server gets many connections for one user.
This wastes server resources.
It can also cause duplicate messages or state conflicts between tabs.

The app must select one tab to hold the live connection.
This tab is the **leader**.
Other tabs are **followers**.
Followers must not hold a live connection.

This is a common frontend system design interview question.
Example: "How does WhatsApp Web stop each open tab from creating its own connection to the chat server?"

The system must answer these questions:

- Which tab is the leader, at any point in time?
- What happens when a tab opens?
- What happens when the leader tab closes normally?
- What happens when the leader tab fails, with no chance to signal a close?
- What happens when two tabs open at almost the same time? Does the system ever show two leaders, even briefly?

## The Solution

This article shows one way to solve the problem, with frontend code only.
The solution does not need a server to run the election.

The solution uses a `SharedWorker`.
A `SharedWorker` is a single script instance. Every tab from the same origin shares this instance.
Each tab opens its own port into this one shared instance.

The shared worker is the single source of truth.
It holds the list of connected tabs.
It runs the election rule.
It tells each tab if the tab is the leader or a follower.

The system has one shared worker. Tabs do not decide the leader on their own.
This design removes the race condition from the election.
The worker processes each connect event and disconnect event one at a time, in order.

### Step 1: Tabs connect to one shared worker

Each tab creates a `SharedWorker` and opens a port to it.
All tabs connect to the same worker instance.
This gives every tab a channel to send messages to, and receive messages from, one shared, central place.

```js
// tab-client.js
const worker = new SharedWorker("worker.js");
const port = worker.port;
port.start();
```

### Step 2: The worker keeps a registry

The worker keeps a list of connected tabs.
Each entry has a unique ID and a connect timestamp.
When a tab connects, the worker adds an entry.
When a tab closes normally, the tab sends a "bye" message.
The worker removes the entry for that tab.

```js
// worker.js
const tabs = new Map(); // id: { port, connectedAt, lastSeen }

onconnect = (event) => {
  const port = event.ports[0];
  port.start();

  // The ID combines a timestamp and a random string. This makes the ID
  // unique within one browser session. It is not a formal UUID.
  const id = `${Date.now()}-${Math.random().toString(36).slice(2)}`;
  tabs.set(id, { port, connectedAt: Date.now(), lastSeen: Date.now() });

  port.postMessage({ type: "welcome", id });
  broadcastLeader();

  port.onmessage = (e) => {
    if (e.data.type === "bye") {
      tabs.delete(id); // The tab sent this event. The worker removes the entry now.
      broadcastLeader();
    } else if (e.data.type === "heartbeat") {
      const entry = tabs.get(id);
      if (entry) entry.lastSeen = Date.now();
    }
  };
};
```

### Step 3: The worker elects a leader

The worker uses one rule: the tab with the earliest connect timestamp is the leader.
The worker applies this rule after each registry change.
The worker sends a message to every connected tab.
This message tells the tab if the tab is the leader or a follower.

Only the worker selects the leader.
Tabs do not select the leader on their own.
This design keeps the leader decision the same across all tabs.

```js
// worker.js
function electLeader() {
  let leaderId = null;
  let earliest = Infinity;
  for (const [id, t] of tabs) {
    if (t.connectedAt < earliest) {
      earliest = t.connectedAt;
      leaderId = id;
    }
  }
  return leaderId;
}

function broadcastLeader() {
  const leaderId = electLeader();
  for (const [id, t] of tabs) {
    t.port.postMessage({ type: "leader", isLeader: id === leaderId, leaderId });
  }
}
```

### Step 4: The worker detects failed tabs

A normal tab close sends a "bye" message.
A failed tab cannot send this message.
The worker needs a second method to detect this case.

Each tab sends a "heartbeat" message every 5 seconds.
The worker records the time of the last heartbeat for each tab.
The worker checks all entries every 5 seconds.
If a tab sends no heartbeat for 15 seconds (3 missed heartbeats), the worker removes the entry for that tab.
The worker applies the election rule again after this removal.

The worker also sends the leader message again every 5 seconds, to every tab, even when nothing changed.
This confirms two facts to each tab: the worker is still active, and the leader status is still correct.
Step 5 shows why this repeated message is necessary.

```js
// worker.js
const HEARTBEAT_INTERVAL = 5000; // 5s
const MISSED_THRESHOLD = 3; // 3 missed beats = ~15s = eviction

setInterval(() => {
  const now = Date.now();
  for (const [id, t] of tabs) {
    if (now - t.lastSeen > HEARTBEAT_INTERVAL * MISSED_THRESHOLD) {
      tabs.delete(id); // The heartbeat timeout triggers this removal.
    }
  }
  broadcastLeader(); // The worker sends this message on every tick, not only on a change.
}, HEARTBEAT_INTERVAL);
```

### Step 5: Each tab checks for old information

A tab can miss a worker message. This can happen when the browser freezes the tab.
When the tab becomes active again, the tab can show old, incorrect information.

Each tab records the time of the last leader message it received.
If 12 seconds pass with no new leader message, the tab stops trusting its own status. The tab waits for a new leader message before it trusts its status again.
The 12-second limit is shorter than the worker's 15-second eviction limit. This order makes sure a tab distrusts old information before the worker would remove the tab from the registry.

```js
// tab-client.js
let lastLeaderMsgAt = Date.now();
const SELF_DISTRUST_THRESHOLD = 12000; // 12s, shorter than the worker's 15s eviction window

port.onmessage = (event) => {
  if (event.data.type === "leader") {
    lastLeaderMsgAt = Date.now();
    // See Step 6 for the leader/follower status logic.
  }
};

setInterval(() => {
  if (Date.now() - lastLeaderMsgAt > SELF_DISTRUST_THRESHOLD) {
    // The tab treats its status as unknown until a new leader message arrives.
  }
}, 2000);
```

### Step 6: Leader status controls the real connection

Only the leader tab opens the real connection to the backend, for example a WebSocket.
Follower tabs do not open this connection.
A follower tab shows a message, for example "connected in another tab."

When a tab's status changes from follower to leader, the tab opens the connection.
When a tab's status changes from leader to follower, the tab closes the connection.
The tab reacts only to a change in status. The tab does not react to every leader message.
This rule stops the tab from opening the connection many times.

```js
// tab-client.js
let isCurrentlyLeader = false;

port.onmessage = (event) => {
  const msg = event.data;
  if (msg.type === "leader") {
    lastLeaderMsgAt = Date.now();

    if (msg.isLeader && !isCurrentlyLeader) {
      connectToBackend();
    } else if (!msg.isLeader && isCurrentlyLeader) {
      disconnectFromBackend();
    }
    isCurrentlyLeader = msg.isLeader;
  }
};
```

The `beforeunload` listener that sends the "bye" message must register one time only, outside `port.onmessage`.
If the listener registers inside `port.onmessage`, the app adds one more listener for each message.
This creates many duplicate listeners.

## Tradeoffs

- **When this pattern shines:** Use this pattern when a live connection has a high cost per connection, or when the server sets a low connection limit per user. Use it when the app usually runs in more than one tab, for example a chat app or a dashboard.
- **When to avoid this pattern:** `SharedWorker` support varies between browsers. Some browsers, including some mobile browsers, do not support `SharedWorker`. If the app must run on these browsers, use a `BroadcastChannel` and `localStorage` lock pattern instead. That pattern runs on more browsers, but the app code must manage the election race itself.
- **What you give up:** The 15-second failure detection window is a design choice. A shorter window detects a failed tab faster. A shorter window can also remove a tab that is only slow, for example a backgrounded tab on a mobile device, or a tab that runs a long synchronous task. A removed tab that is still active can find that another tab replaced it as leader.
- **Extra complexity to plan for:** A `SharedWorker` instance does not reload when its script file changes. The old instance keeps running until every connected tab closes. A developer can edit `worker.js`, reload one tab, and still run the old code. To pick up a change, close every tab connected to the worker, or stop the worker instance from the browser's worker inspector (`chrome://inspect/#workers` in Chrome, `about:debugging#/runtime/this-firefox` in Firefox), then reload.

## Out of Scope

This article answers one question: which tab is the leader?

This article does not answer a second question: how do follower tabs receive live data, for example new chat messages?

A real app like WhatsApp Web must also solve this second question.
One method: the leader tab relays each message through the shared worker to all follower tabs.
This method adds more complexity. The shared worker becomes a message relay, not only an election system.

This article does not include this method.
This article keeps its scope to the leader election problem only.

## Key Takeaways

- A `SharedWorker` gives the app one central process to decide the leader. This removes the leader election race condition.
- The system needs two different methods to detect a lost tab: a "bye" message for a normal close, and a heartbeat timeout for a failed tab.
- A tab must react only to a change in leader status, not to every leader message. This rule stops the tab from opening or closing its connection many times.
- A tab must distrust its own old status after a time limit shorter than the worker's eviction limit. This rule stops a tab from acting on old information after the browser reactivates the tab.
- This pattern solves one problem: which tab owns the connection. This pattern does not solve how follower tabs receive the data from that connection.

## References

- [MDN: SharedWorker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)
- [MDN: BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)
- [Can I use: SharedWorker](https://caniuse.com/sharedworkers)
