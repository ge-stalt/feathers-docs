# Scaling

Depending on your requirements, your feathers application may need to provide high availability. Feathers is designed to scale.

The types of [providers](../providers/README.md) used in a feathers application will impact the scaling configuration. For example, a feathers app that uses the `feathers-rest` adapter exclusively will require less scaling configuration because HTTP is a stateless protocol. If using websockets (a stateful protocol) through the `feathers-socketio` or `feathers-primus` adapters, configuration may be more complex to ensure websockets work properly.

## Horizontal Scaling

Scaling horizontally refers to either:

- setting up a [cluster](https://nodejs.org/api/cluster.html), or
- adding more machines to support your application

To achieve high availability, varying combinations of both strategies may be used.

## Cluster configuration

[Cluster](https://nodejs.org/api/cluster.html) support is built into core NodeJS. Since NodeJS is single threaded, clustering allows you to easily distribute application requests among multiple child processes (and multiple threads). Clustering is a good choice when running feathers in a multi-core environment.

Below is an example of adding clustering to feathers with the `feathers-socketio` provider. By default, websocket connections begin via a handshake of multiple HTTP requests and are upgraded to the websocket protocol. However, when clustering is enabled, the same worker will not process all HTTP requests for a handshake, leading to HTTP 400 errors. To ensure a successful handshake, force a single worker to process the handshake by disabling the http transport and exclusively using the `websocket` transport.

There are notable side effects to be aware of when disabling the HTTP transport for websockets. While all modern browsers support websocket connections, there is no websocket support for [IE <=9 and Android Browser <=4.3](http://caniuse.com/#feat=websockets). If you must support these browsers, use alternative scaling strategies.

```js
import cluster from 'cluster';
import feathers from 'feathers';
import socketio from 'feathers-socketio';

const CLUSTER_COUNT = 4;

if (cluster.isMaster) {
  for (let i = 0; i < CLUSTER_COUNT; i++) {
    cluster.fork();
  }
} else {
  const app = feathers();
  // ensure the same worker handles websocket connections
  app.configure(socketio({
    transports: ['websocket']
  }));
  app.listen(4000);
}
```

In your feathers client code, limit the socket.io-client to the `websocket` transport and disable `upgrade`.

```js
import hooks from 'feathers-hooks';
import feathers from 'feathers/client';
import io from 'socket.io-client';
import socketio from 'feathers-socketio/client';

const app = feathers()
  .configure(hooks())
  .configure(socketio(
    io('http://api.feathersjs.com', {
      transports: ['websocket'],
      upgrade: false
    })
  ));
```
