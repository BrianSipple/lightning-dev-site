---
layout: page
title: How to write a Javascript gRPC client for the Lightning Network Daemon

---

### Setup and Installation

First, you'll need to initialize a simple nodejs project:
```
npm init (or npm init -f if you want to use the default values without prompt)
```

Then you need to install the Javascript grpc library dependency:
```
npm install grpc --save
```

You also need to copy the `lnd` `rpc.proto` file in your project directory (or
at least somewhere reachable by your Javascript code).

The `rpc.proto` file is [located in the `lnrpc` directory of the `lnd`
sources](https://github.com/lightningnetwork/lnd/blob/master/lnrpc/rpc.proto).

In order to allow the auto code generated to compile the protos successfully,
you'll need to comment out the following line:
```
//import "google/api/annotations.proto";
```

#### Imports and Client

Every time you work with Javascript gRPC, you will have to import `grpc`, load
`rpc.proto`, and create a connection to your client like so:

```js
var grpc = require('grpc');
var lnrpcDescriptor = grpc.load("rpc.proto");
var lnrpc = lnrpcDescriptor.lnrpc;
var lightning = new lnrpc.Lightning('localhost:10009', grpc.credentials.createInsecure());
```

### Examples

Let's walk through some examples of Javascript gRPC clients.

#### Simple RPC

```js
> lightning.getInfo({}, function(err, response) {
  	console.log('GetInfo:', response);
  });
```

You should get something like this in your console:

```
GetInfo: { identity_pubkey: '03c892e3f3f077ea1e381c081abb36491a2502bc43ed37ffb82e264224f325ff27',
  alias: '',
  num_pending_channels: 0,
  num_active_channels: 1,
  num_peers: 1,
  block_height: 1006,
  block_hash: '198ba1dc43b4190e507fa5c7aea07a74ec0009a9ab308e1736dbdab5c767ff8e',
  synced_to_chain: false,
  testnet: false,
  chains: [ 'bitcoin' ] }
```

#### Response-streaming RPC

```js
var call = lightning.subscribeInvoices({});
call.on('data', function(invoice) {
    console.log(invoice);
})
.on('end', function() {
  // The server has finished sending
})
.on('status', function(status) {
  // Process status
  console.log("Current status" + status);
});
```
Now, create an invoice for your node at `localhost:10009` and send a payment to
it. Your Javascript console should display the details of the recently satisfied
invoice.

#### Bidirectional-streaming RPC

This example has a few dependencies:
```shell
npm install --save async lodash bytebuffer
```

You can run the following in your shell or put it in a program and run it like
`node script.js`

```js
// Load some libraries specific to this example
var async = require('async');
var _ = require('lodash');
var ByteBuffer = require('bytebuffer');

var dest_pubkey = <RECEIVER_ID_PUBKEY>;
var dest_pubkey_bytes = ByteBuffer.fromHex(dest_pubkey);

// Set a listener on the bidirectional stream
var call = lightning.sendPayment();
call.on('data', function(payment) {
  console.log("Payment sent:");
  console.log(payment);
});
call.on('end', function() {
  // The server has finished
  console.log("END");
});

// You can send single payments like this
call.write({ dest: dest_pubkey_bytes, amt: 6969 });

// Or send a bunch of them like this
function paymentSender(destination, amount) {
  return function(callback) {
    console.log("Sending " + amount + " satoshis");
    console.log("To: " + destination);
    call.write({
      dest: destination,
      amt: amount
    });
    _.delay(callback, 2000);
  };
}
var payment_senders = [];
for (var i = 0; i < 10; i++) {
  payment_senders[i] = paymentSender(dest_pubkey_bytes, 100);
}
async.series(payment_senders, function() {
  call.end();
});

```
This example will send a payment of 100 satoshis every 2 seconds.

### Conclusion

With the above, you should have all the `lnd` related `gRPC` dependencies
installed locally in your project. In order to get up to speed with `protofbuf`
usage from Javascript, see [this official `protobuf` reference for
Javascript](https://developers.google.com/protocol-buffers/docs/reference/javascript-generated).
Additionally, [this official gRPC
resource](http://www.grpc.io/docs/tutorials/basic/node.html) provides more
details around how to drive `gRPC` from `node.js`.

