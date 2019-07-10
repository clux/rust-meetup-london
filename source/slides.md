<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

### Rust on kubernetes

- Eirik Albrigtsen : [github.com/clux](https://github.com/clux)
- Babylon Health : [github.com/Babylonpartners](https://github.com/Babylonpartners)

<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 560px" />

NOTES:
- Hi. Eirik. Platform, lot of things related to kube and rust.
- testing, building, and pushing dockerised rust apps
- Deploying to kube (largely glossing over this)
- Talking to the kubernetes API and extending it with your own custom resources

---
<!-- .slide: data-background-color="#353535" -->
Building

- Cargo.lock
- cargo build
- dockerise

notes:
- why mention? cargo build + docker?

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

alpine

<ul>
  <li class="fragment">musl-libc</li>
  <li class="fragment">busybox</li>
</li>

notes:
- musl-libc is a rewrite of glibc, smaller, more performant (some benchmarks)
- 10-30% performance improvement suggested (but bench for your case..)
- fewer code paths, less of the specific glibc legacy, but more edge cases (locales, glibc specifics)
- busybox: single binary version of coreutils(TODO: ?)
- together: alpine; 5MB linux distro
- don't pick scratch, 5MB small price to pay for bash and a working package manager
- don't pick ubuntu unless you need to (order of magnitude more disk space)
- distroless; sure, it's basically equivalent to...

---
<!-- .slide: class="color" style="min-width: 100%" -->
normal image

```dockerfile
FROM alpine:3.9
RUN apk --no-cache add ca-certificates
COPY ./controller /bin/
EXPOSE 8080
ENTRYPOINT ["/bin/controller"]
```

notes:
- always pin your deps (otherwise why even lockfile)
- dont apk upgrade (3.9 tag updates should receive security upgrades)
- ca-certs if you need ssl/rusttls (if you have a service mesh, you might not)
- but might need to install anyway for non-http conn (ssl postgres -> psql with --with-openssl)
- import certs from aws to use secure ssl rds
- rest is just like scratch
- static compilation so (libc + ssl + psql linked into entrypoint)
- designed for one binary per container (otherwise you duplicate C deps within container)

---
compiling for musl

```sh
rustup target add x86_64-unknown-linux-musl
cargo build --target=x86_64-unknown-linux-musl --release
```

notes:
- easiest locally, basic cross compilation support in rust already
- but you may find it doesn't work depending on what dependencies you need
- psql (or other sql client)
- curl (but not needed anymore due to hyper), openssl (hyper ssl support, alternatives underway)
- zlib (serving anything gzipped)
- any other library that has C bindings
---
compiling for musl (with C dependencies)

```sh
docker run --rm \
  -v $PWD:/volume \
  -t clux/muslrust:stable \
  cargo build --release
```

notes:
- one of 3-4 big musl build images (this is the only one that builds continually and makes descriptive tags)
- push like 20GB a month from travis for free for this
- my image, so you may want to fork it in a company.. really rust should support it (but no feedback on that)
- (curl, openssl, psql, zlib, sqlite)
- there are more general cross compile (cross, xargo, embedded), but for cloud: musl 64 bit linux FTW.

---
ci flow

```
1. cargo test
1. cargo build --release (then stash musl binary)
2. github releases (attach musl binary to release on tag build)
2. docker build (attach musl binary) && docker push
```
TODO: replace with shipcat tag flow?

notes:
- why not multistage? caching + releasing
- caching -> open docker issue on caching build dir
- releasing -> separate thing you can do with a binary (if building a CLI tool)
- what is multistage except removing that possibility for a tiny encapulation win?

---
caching (circleci)

```yaml
  musl_build:
    docker: [ { image: clux/muslrust:stable-1.36.0 } ]
    steps:
      - checkout
      - restore_cache:
          keys: ["reg-{{ .Environment.CACHE_VERSION }}"]
      - restore_cache:
          keys: ["targ-{{ .Environment.CACHE_VERSION }}"]
      - run: cargo build --release
```
```yaml
      - save_cache:
          key: targ-{{ .Environment.CACHE_VERSION }}
          paths: ["target"]
      - save_cache:
          key: reg-{{ .Environment.CACHE_VERSION }}
          paths: ["/root/.cargo"]
      - run: mv target/MUSLTRIPLE/release/app{,.MUSLTRIPLE}
      - persist_to_workspace:
          root: target/MUSLTRIPLE/release/
          paths: ["app.MUSLTRIPLE"]
```


notes:
- {TRIPLE} = `x86_64-unknown-linux-musl`
- minimal circle steps with caching
- caches two things: registry (DOWNLOADING CRATE), target (COMPILING CRATE)
- lets you fiddle the cache version. why? `sha(.lock)` is inefficient depending on release cadence
- that is recommended though

---
flow (circleci)

```yaml
    jobs:
      - cargo_test:
          filters: { branches: { ignore: ["master"] } }
      - mac_build:
          filters:
            tags: { only: /^.*/ }
            branches: { only: ["master"] }
      - musl_build:
          filters: { tags: { only: /^.*/ } }
```
```yaml
      - docker_build:
          requires: ["musl_build"]
          filters: { branches: { only: ["master"] } }
      - github_release:
          requires:
            - musl_build
            - mac_build
          filters:
            tags: { only: /^.*/ }
            branches: { ignore: /.*/ }
```

notes:
- abbreviating awkward circle yaml a bit here (general idea, but should be valid)
- PR build from branch: (cargo test + musl build)
- master merge: (musl build + docker build)
- github release: (musl + mac build then docker build + github_release)

- test + build in musl? may as well.

---
<!-- .slide: data-background-color="#353535" -->
Kubernetes

breather slide.. images.. memes

---
<!-- .slide: data-background-color="#353535" -->
Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-controller
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: foo-controller
        image: "clux/controller:0.2.1"
        imagePullPolicy: IfNotPresent
```
<p class="fragment">..[clux/controller-rs](https://github.com/clux/controller-rs)</p>

notes:
- missing all the things: resource requests/limits, readinessProbe, evars, service account, ports, labels
- framework choice. hyper or actix. (actix benches are awesome but futures learning curve)
- rocket is super nice, but still synchronous

---
<!-- .slide: data-background-color="#353535" -->
Connecting to kubernetes api

```rust
let cfg = match env::var("HOME").expect("have HOME dir").as_ref() {
    "/root" => kube::config::incluster_config(),
    _ => kube::config::load_kube_config(),
}.expect("Failed to load kube config");
let client = kube::client::APIClient::new(cfg)?;
```

notes:
- switch for local dev (cargo run)
- local dev might not work if you have auth hooks or oidc providers (can impersonate) TODO: link!
- TODO: why root in container?!

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Using the kube api

```rust
let pods = Api::v1Pod(client).within("default");
let blog = pods.get("blog")?;
println!("blog has containers: {:?}", blog.spec.containers);

for p in pods.list(&ListParams::default())? {
    println!("found pod", p.name);
    pods.delete(p.name)?;
}
```

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Under the hood

TODO: replace with image!
```rust
impl<K> Api<K> where
    K: Clone + DeserializeOwned + KubeObject
{
  fn get(&self, name: &str)
    -> Result<K>;
  fn create(&self, pp: &PostParams, data: Vec<u8>)
    -> Result<K>;
  fn patch(&self, name: &str, pp: &PostParams, patch: Vec<u8>)
    -> Result<K>;
  fn replace(&self, name: &str, pp: &PostParams, data: Vec<u8>)
    -> Result<K>;
  fn watch(&self, lp: &ListParams, version: &str)
    -> Result<Vec<WatchEvent<P, U>>>;
  fn list(&self, lp: &ListParams)
    -> Result<ObjectList<K>>;
  fn delete_collection(&self, lp: &ListParams)
    -> Result<Either<ObjectList<K>, Status>>;
  fn delete(&self, name: &str, dp: &DeleteParams)
    -> Result<Either<K, Status>>;

  fn get_status(&self, name: &str)
    -> Result<K>;
  fn patch_status(&self, name: &str, pp: &PostParams, patch: Vec<u8>)
      ->Result<K>;
  fn replace_status(&self, name: &str, pp: &PostParams, data: Vec<u8>)
      -> Result<K>;
}
```

---
<!-- .slide: data-background-color="#353535" -->
Using the kubernetes api

```rust
let f = json!({
    "apiVersion": "clux.dev/v1",
    "kind": "Foo",
    "metadata": { "name": "baz" },
    "spec": { "name": "baz", "info": "baz info" },
});
let o = foos.create(&pp, serde_json::to_vec(&f)?)?;
assert_eq!(f["metadata"]["name"], o.metadata.name)
```
notes:
- kube crate (there's 3, this is _the one_ that tries to do a simplified api and higher level concepts)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Using the kubernetes api post/patch

```rust
let patch = json!({
    "spec": { "info": "patched baz" }
});
let o = foos.patch("baz", &pp, serde_json::to_vec(&patch)?)?;
assert_eq!(o.spec.info, "patched baz");
assert_eq!(o.spec.name, "baz");
```

notes:

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Controller patterns

```fortran
10 WATCH resource
20 REACT to modified/added/deleted events
30 GOTO 10
```

notes:
- conceptually simple, all the meat should be in in REACTING to the events
- but to watch, you have to tell kube where to watch from
- a watch query always blocks for the given wait time, even if there are events
- with every return, you are told, where to resume from

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Higher level abstractions

<ul>
  <li class="fragment">`Informer<K>`</li>
  <li class="fragment">`Reflector<K>`</li>
</li>

notes:
- Informer: lets you handle raw events as they come in
- Use case: make changes to it/other resources when events occur (controller pattern)
- Ref: maintains an in memory cache of a ressource K (only REACTION)
- Updates when it polls, with a RwLock, and lets you read whenever.
- Use case: monitoring a resource, presenting an api around many resources..
- (Can build a Ref from an Inf by just using events to update a local Map).

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Handling events via informers

```rust
match ev { // ev : WatchEvent<Pod>
    WatchEvent::Added(o) => {
        info!("Added Pod: {}", o.metadata.name);
    },
    WatchEvent::Modified(o) => {
        info!("Modified Pod: {}", o.metadata.name);
    },
    WatchEvent::Deleted(o) => {
        info!("Deleted Pod: {}", o.metadata.name);
    },
    WatchEvent::Error(e) => {
        warn!("Error event: {:?}", e);
    }
}
```

notes:
- go you attach event handlers to the informers
- rust we drain an internal queue and get events moved to us
- o is complete object (Pod object here)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
why make your own controller

breather slide...

notes:
- kube has 100 of them and they all reconcile the state
- a controller can keep track of state in status, and work towards reconciling
- its a nicer model than config management monorepo (for things that fit the cluster model)
- old model: having N CI jobs per cluster, triggering them periodically, implementing diffing (do i need to do something) yourself, reconcile state in the job
- new model: write one tiny app, in a way that's meant to deal with events, deploy it to all clusters

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
what is an object

```rust
#[derive(Deserialize, Serialize, Clone)]
pub struct Object<T, U> where T: Clone, U: Clone
{
    #[serde(flatten)]
    pub types: TypeMeta,
    pub metadata: ObjectMeta,
    pub spec: T,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub status: Option<U>,
}
```

notes:
- each object composed of spec (what you want) and status (what it is)
- crucially, kube lets you extend the api as long as you define T and optionally, U

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Custom Resources

```rust
use kube::api::Object;

#[derive(Deserialize, Serialize, Clone)]
pub struct FooSpec {
    name: String,
    info: String,
}

#[derive(Deserialize, Serialize, Clone, Debug, Default)]
pub struct FooStatus {
    isBad: bool,
}

type Foo = Object<FooSpec, FooStatus>;
```

notes:
- in go you need the external codegen for this, but in rust it's all generics
- write the struct, derive our traits
- (pass some initial yaml to kube api server, ~30 lines generally)

---
<!-- .slide: data-background-image="./guesswork.gif" data-background-size="100% auto" style="color: #2fa;" -->
Using the CRD api

```rust
let foos : Api<Foo> = Api::customResource(client, "foos")
    .version("v1")
    .group("clux.dev")
    .within("default");

let baz = foos.get("baz")?;
assert_eq!(baz.spec.info, "baz info");
```

notes:
- just like native objects (but need to specify version + group (yours))

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Using the CRD api

```rust
let foos = Api::customResource("foos")
    .version("v1")
    .group("clux.dev")
    .within(&namespace);
let foo_inf = Informer::new(client.clone(), foos)
    .timeout(30)
    .init()?;
```
notes:
- can make informers + reflectors on them
- from there you can basically just start handling events
- and ensuring that you can reconcile changes

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

what

breather slide

notes:
- question on using kube api

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->



---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

---
<!-- .slide: data-background-image="./applause.gif" data-background-size="100% auto" class="color"-->

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

- Eirik - [github.com/clux](https://github.com/clux) - [clux.dev](https://clux.dev)
- Babylon Health - [babylonhealth.com](https://babylonhealth.com)

<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 560px" />
