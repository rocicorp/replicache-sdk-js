# Replicache JS SDK

### Realtime Sync for Any Backend Stack

![Node.js CI](https://github.com/rocicorp/replicache-sdk-js/workflows/Node.js%20CI/badge.svg)

## 👋 Quickstart

This document walks you through getting Replicache up and running in your own application, talking to your own backend.

## 🏃‍♂️ Create your Client View endpoint

Add a `/replicache-client-view` endpoint to your API server that responds to HTTP `POST` and with the data you want cached on the device, like so:

```
{
  "lastMutationID": 0,
  "clientView": {
    "/todo/1": {
      "title": "Take out the trash",
      "description": "Don't forget to pull it to the curb.",
      "done": false,
    },
    "/todo/2": {
      "title": "Pick up milk",
      "description": "2%",
      "done": true,
    }
  }
}
```

You can put any key/value pairs you want in the `"clientView"` section.

For more details on implementing your Client View, see [Client View Setup](https://github.com/rocicorp/replicache/blob/stable/SERVER_SETUP.md#step-1-downstream-sync).

You can also skip this step for now, and just use our sample backend instead. See [replicache-echo-backend](https://github.com/rocicorp/replicache-echo-backend).

## 🛹 Install Replicache

In your web client root:

```
npm install replicache
```

## 🚴‍♂️ Start Diff Server

Start this and keep it running:

```
./node_modules/replicache/bin/diff-server  --disable-auth --db=/tmp/diffs-db --account-db=/tmp/accounts-db serve
```

**Note:** This step is temporary - [the diff server is being removed from the Replicache sync protocol](https://www.notion.so/Differential-Client-View-12be3a636c9f404b88d49ecbd100a694). Yay!

## 🚗 User Interface

Now let's add a user interface to render the synced data.

### If you use React:

1. `npm add replicache-react-util`
1. If you use webpack 4, follow [this procedure](./webpack4-setup.md) to configure webpack to load wasm properly. If you use webpack 5, nothing special is needed. If you use another bundler, try it and let us know 😀.
1. Then, add this code somewhere:

```js
import React from 'react';
import ReactDOM from 'react-dom';
import Replicache from 'replicache';
import {useSubscribe} from 'replicache-react-util';

const rep = new Replicache({
  // The URL your Client View from above is running at.
  clientViewURL: 'http://localhost:3001/replicache-client-view',

  // Auth token required for your Client View endpoint, if any.
  dataLayerAuth: '<authtoken>',

  // The URL your Diff Server from above is running at, defaults to
  // localhost:7001.
  diffServerURL: 'http://localhost:7001/pull',

  // Auth Token to use for the Diff Server. For development, always
  // use 'sandbox'.
  diffServerAuth: 'sandbox',

  // Path to where Replicache wasm module can be found.
  // Only needed if you use webpack 4.
  wasmModule: '/node_modules/replicache/out/replicache.wasm',
});

function MyApp() {
  const entries = useSubscribe(rep, async tx => await tx.scanAll(), []);

  return (
    <table border="1">
      <tbody>
        {entries.map(([k, v]) => (
          <tr key={k}>
            <td>{k}</td>
            <td>{JSON.stringify(v)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

var elm = document.createElement('div');
document.body.appendChild(elm);
ReactDOM.render(React.createElement(MyApp), elm);
```

### Vanilla JS:

```html
<script type="module">
  import Replicache from './node_modules/replicache/out/replicache.dev.js';

  const rep = new Replicache({
    // The URL your Client View from above is running at.
    clientViewURL: 'http://localhost:3001/replicache-client-view',

    // Auth token for your Client View, if any.
    dataLayerAuth: '<authtoken>',

    // The URL your Diff Server from above is running at, defaults to
    // localhost:7001.
    diffServerURL: 'http://localhost:7001/pull',

    // Auth Token to use for the Diff Server. For development, always
    // use 'sandbox'.
    diffServerAuth: 'sandbox',
  });

  rep.subscribe(async tx => await tx.scanAll(), {
    onData: result => {
      const tbody = document.querySelector('tbody');
      tbody.innerHTML = '';
      for (const [k, v] of result) {
        const row = document.createElement('tr');
        const td1 = document.createElement('td');
        const td2 = document.createElement('td');
        td1.textContent = k;
        td2.textContent = JSON.stringify(v);
        row.appendChild(td1);
        row.appendChild(td2);
        tbody.appendChild(row);
      }
    },
  });
</script>
<table border="1">
  <tbody></tbody>
</table>
```

Replicache polls your Client View on a timer and updates the UI automatically.

Notes:

- In production, you can use our hosted Diff Server instead of a local one. See [Create Hosted Diff Server Account](https://github.com/rocicorp/replicache/blob/stable/SERVER_SETUP.md#if-your-client-view-is-publicly-accessible) for details.
- You can adjust the frequency of the polling using the [syncInterval](https://js.replicache.dev/interfaces/replicacheoptions.html#syncinterval) property. However, polling is typically only used for development. In production, we recommend using a [WebSocket poke](TODO) to tell the client when to sync.

## 🏎 Batch Endpoint

If you are using `replicache-echo-backend` you can skip this step!

Otherwise, add a `/replicache-batch` endpoint to your API server that accepts and processes batches of mutations that look like this:

```
{
  "clientID": "CB94867E-94B7-48F3-A3C1-287871E1F7FD",
  "mutations": [
    {
      "id": 7,
      "name": "createItem",
      "args": {
        "key": "create-item-0.4420193",
        "value": "foo",
      }
    },
    {
      "id": 8,
      "name": "createItem",
      "args": {
        "key": "create-item-0.21301",
        "value": "bar",
      }
    },
  ]
}
```

See [Replicache Upstream Sync](https://github.com/rocicorp/replicache/blob/stable/SERVER_SETUP.md#step-4-upstream-sync) for instructions on how to implement this endpoint.

### 🛫 Client-Side Mutations

Add the following code to the end of your script block in your UI.

```js
const createItem = rep.register('createItem', async (tx, {key, value}) => {
  await tx.put(key, value);
});

var button = document.createElement('button');
button.textContent = 'New item...';
document.body.appendChild(button);
button.onclick = () => {
  createItem({key: `new-item-${Math.random()}`, value: 'foo'});
};
```

### 🚀 Server-Side Mutations

Wire up your batch handler:

```patch
const rep = new Replicache({
   ...
+  // The URL to your batch endpoint added above.
+  batchURL: 'http://localhost:3001/replicache-batch',
});
```

When you tap the button the change happens immediately on the client, but is
momentarily later sent to the server. You can verify it by turning the network
off in web inspector or slowing it down.

## Next Steps

That's it! You've built a fully-functioning Replicache-powered app against your own backend. What will you do next?

- [Check out a full React-based todo sample](https://github.com/rocicorp/replicache-sdk-js/tree/main/sample/redo)
- [Browse the full JS documentation](https://replicache-sdk-js.now.sh/)
