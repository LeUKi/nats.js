# Runtimes

The NATS client for JavaScript supports many standard runtimes out of the box:

- Node.js/Bun
- Deno
- Browser

The runtime is distributed in two different registries:

- npmjs.com - these are npm bundles that target Node.js and compatible runtimes
  (Bun)
- jsr.io - these are esm versions (typically targetting Deno or browser)

Note that the names of the bundles are the same in the two registries
`@nats-io/<module>`. The reason for this is that code that is properly written
will look the same and be compatible across the runtimes provided no additional
user-added dependencies prevent this. The module manifest
(package.json/deno.json) will provide the right mapping to the registry, your
code simply references the correct library regardless of registry.

Let's make a project for your favorite runtime, Note that this example is
intended to create a simple development environment, not to explain NATS
concepts, etc. For that refer to the [README.md](README.md).

## Setting Up a Dev Environment

### Node.js

```bash
mkdir -p nats-dev/node
cd nats-dev/node
npm init esnext -y
# the tsx module allows to run typescript files directly by doing `tsx filename.ts`
npm install @nats-io/transport-node tsx
```

### Bun

```bash
mkdir nats-dev/bun
cd nats-dev/bun
bun init -y
bun add @nats-io/transport-node
```

### Deno

```bash
mkdir nats-dev/deno
cd nats-dev/deno
deno init
deno add jsr:@nats-io/transport-deno
```

## Let's add a simple program that we can run

Added to the `nats-dev/<runtime>/index.ts` you used above.

If using Deno, uncomment the second line, and comment out the first.

```typescript
import { connect, deferred, nuid } from "@nats-io/transport-node";
// import { connect, deferred, nuid } from "@nats-io/transport-deno";

const nc = await connect({ servers: "demo.nats.io" });
console.log(`connected`);

const subj = nuid.next();

nc.subscribe(subj, {
  callback: (err, msg) => {
    console.log(msg.subject, msg.json());
  },
});

let i = 0;
const d = deferred();
const timer = setInterval(() => {
  i++;
  nc.publish(subj, JSON.stringify({ ts: new Date().toISOString(), i }));
  if (i === 10) {
    clearInterval(timer);
    d.resolve();
  }
}, 1000);

await d;
await nc.drain();
```

If you want to use a WebSocket connection, replace `await connect()` as follows:

```typescript
import { wsconnect } from "@nats-io/transport-node";
// import { wsconnect } from "@nats-io/transport-deno";
const nc = await wsconnect({ servers: "demo.nats.io:8443" });
```

Save it to `index.ts`

## To run:

```bash
# Nodejs
tsx index.ts

# Bun
bun index.ts

# Deno
deno run -A index.ts
```

You should see some output similar to:

```bash
connected
29CODPRD5FJ0INNDXBAOC2 {
  ts: "2024-11-13T17:59:22.853Z",
  i: 1,
}
29CODPRD5FJ0INNDXBAOC2 {
  ts: "2024-11-13T17:59:23.852Z",
  i: 2,
}
...
```

Congratulations. You have a working NATS project. Now go explore the
documentation.

## Running in the Browser

As discussed above, out of the box, the `@nats-io/nats-core` module provides a
W3C WebSocket transport, that you can use directly from Node.js, Bun, Deno and
most importantly in a Browser via the `wsconnect()` function.

The setup is similar as you have seen above, with the exception that you have to
import and configure your Web frameworks libraries, which is beyond the scope of
this document.

As samples, take a look at my examples for Next.js and React Native pointed to
below. The base projects are generated by the stand tools adding a few
components but showcase connection management, simple pub/sub, kv and object
store monitoring.

Note that only latest versions of the frameworks are known to be compatible, and
while I know that the samples work, but they may not if you have more complex
setups (JavaScript ecosystem is an interesting world.).

### CDN

A number of free CDNs are available that make it possible to reference libraries
distributed via NPM accessible as ESM modules by a simple URL. One such CDN is
[jsdelivr](https://www.jsdelivr.com/):

```javascript
import { wsconnect } from "https://esm.run/@nats-io/nats-core";
import { jetstreamManager } from "https://esm.run/@nats-io/jetstream";
import { Kvm } from "https://esm.run/@nats-io/kv";
import { Objm } from "https://esm.run/@nats-io/obj";
import { Svcm } from "https://esm.run/@nats-io/services";

// Add your code here
```

### Next.js

Is supported out of the box - if you want to
[see an example, checkout this repo](https://github.com/aricart/nats-nextjs-example)

### React Native

Previous versions of the libraries were difficult to use in React Native
requiring shims and many workarounds to get going. Well, we happy to say that it
works out of the box in the latest version of ReactNative:

```bash
# Run the template generator
npx create-expo-app@latest
# React native is now starting to support package exports
# https://reactnative.dev/blog/2023/06/21/package-exports-support#enabling-package-exports-beta
# set 'resolver.unstable_enablePackageExports = true'

npx expo customize metro.config.js
```

Edit `metro.config.js`:

```js
// Learn more https://docs.expo.io/guides/customizing-metro
const { getDefaultConfig } = require("expo/metro-config");

/** @type {import('expo/metro-config').MetroConfig} */
const config = getDefaultConfig(__dirname);
config.resolver = config.resolver || {};
config.resolver.unstable_enablePackageExports = true;

module.exports = config;
```

That is it - now you can just import `@nats-io/nats-core` and use the WebSocket
client right out of the box. JetStream, KV and ObjectStore all work as well. For
a starting point sample
[take a look at this repo](https://github.com/aricart/nats-react-native)

## Esbuild

While you can [use a CDN](#CDN) to use the libraries in your browser, sometimes
it is more convenient to package and host your own, here's an example using
`esbuild`:

```bash
# install esbuild
npm install esbuild --global
```

```bash
mkdir mynats
cd mynats
npm init es6 -y
# Adding all the nats libraries here, but you can certainly reference the ones
# you need
npx jsr @nats-io/nats-core
npx jsr @nats-io/jetstream
npx jsr @nats-io/kv
npx jsr @nats-io/obj
npx jsr @nats-io/services
```

Now creating a simple file that re-exports all the libraries, "nats.js"

```javascript
export * from "@nats-io/nats-core";
export * from "@nats-io/jetstream";
export * from "@nats-io/kv";
export * from "@nats-io/obj";
export * from "@nats-io/services";
```

Create your own local bundled version of the libraries:

```bash
esbuild --format=esm --bundle --minify nats.js > nats.mjs
```

Create your own NATS application referencing the bundled modules (as `index.js`):

```javascript
import { jetstreamManager, Kvm, Objm, Svcm, wsconnect } from "./nats.mjs";

const nc = await wsconnect({ servers: "demo.nats.io:8443" });
console.log(`connected: ${nc.getServer()}`);

const jsm = await jetstreamManager(nc);
let streams = 0;
for await (const si of jsm.streams.list()) {
  streams++;
}
console.log(`found ${streams} streams`);

let kvs = 0;
const kvm = new Kvm(nc);
for await (const k of kvm.list()) {
  kvs++;
}
console.log(`${kvs} streams are kvs`);

let objs = 0;
const objm = new Objm(nc);
for await (const k of objm.list()) {
  objs++;
}
console.log(`${objs} streams are object stores`);

let services = 0;
const svm = new Svcm(nc);
const c = svm.client();
for await (const s of await c.ping()) {
  services++;
}
console.log(`${services} services were found`);

await nc.close();
```

Here's a run of the above in Deno, which is acting just like a browser
JavaScript runtime:

```bash
deno run -A index.js
connected: demo.nats.io:8443
found 278 streams
155 streams are kvs
27 streams are object stores
0 services were found
```

or in th browser directly (note the nats.mjs is referenced relative of the
script file):

```html
<html>
  <body>
    <script type="module" src="./index.js" />
  </body>
</html>
```
