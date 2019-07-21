<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

### Rust on kubernetes

- Eirik Albrigtsen : [github.com/clux](https://github.com/clux) - [clux.dev](https://clux.dev)
- Babylon Health : [github.com/Babylonpartners](https://github.com/Babylonpartners)

<img src="./babylon.png" style="background: none; box-shadow: none; border: none; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="background: none; box-shadow: none; border: none; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="background: none; box-shadow: none; border: none; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="background: none; box-shadow: none; border: none; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="background: none; box-shadow: none; border: none; width: 120px; position: absolute; top: 300px; left: 560px" />

NOTES:
- Hi. Eirik. Platform, lot of things related to kube and rust.
- Writing kube controllers (app managing resources in kube), and extending the kubernetes API with your own CRs, and writing controllers for that.
- but first some operational basic, managing microsvcs in rust

---
<!-- .slide: data-background-image="./fry-safety.webp" data-background-size="100% auto" class="color"-->

notes:
- safety dance when developing in new lang:
- lockfiles, pin deps, run tests, lint
- ez coz: build system
- BUT dockerising, slightly tricky

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
docker

<ul>
  <li class="fragment">[FROM rust:1.36-stretch](https://hub.docker.com/_/rust)</li>
  <li class="fragment">[travis-ci/languages/rust](https://docs.travis-ci.com/user/languages/rust/)</li>
  <li class="fragment">[circleci docker builds](https://circleci.com/docs/2.0/building-docker-images/)
</li>

notes:
- not tricky because no official support...
- official mstage images, travis official support, circleci docker.. BUT

- travis -> off supp outside docker, caching
- circle docker vs non-docker cmds
- circle examples (even if super error-prone, quirky yaml)


---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
caching

<ul>
<li class="fragment">1GB cache for 10MB artifact</li>
<li class="fragment">[docker build does not support cached input](https://github.com/moby/moby/issues/7665)</li>
<li class="fragment">[circleci buildkit support](https://ideas.circleci.com/ideas/CCI-I-1003) - [buildkit orb](https://circleci.com/orbs/registry/orb/springload/buildkit)</li>
</ul>

notes:
- motivation: rust sizes.. 10min build vs 30s build
- docker build: closed issue on caching build dir (rabbit hole link)
- multistage: same issue
- buildkit: better replacement in future
- => docker run with volume mount and caching the directory

---
<!-- .slide: data-background-image="./zoid-speed.webp" data-background-size="100% auto" class="color"-->

<img src="./gnu.png" style="float: left; background: none; box-shadow: none; border: none; position: absolute; right: 0px" />
<img src="./alpine.png" style="float: left; filter: brightness(2); background: none; box-shadow: none; border: none; position: absolute;" />
<img src="./distroless.png" style="float: left; background: none; box-shadow: none; border: none; position: absolute; left: 0px" />

notes:
- ubuntu vs alpine vs distroless:static
- least privilege principle (sensible argument)
- benchmark / speed argument of libc choice (contradictory, measure)
- last 2 => compile for musl
- musl-libc is an 8yo libc inmpl (smaller than glibc, static link, more performant (some benchmarks 10-30% but bench)
- reuseable on all distros (not true other way)


---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

alpine

<ul>
  <li class="fragment"><a href="https://www.musl-libc.org/intro.html">musl-libc</a></li>
  <li class="fragment"><a href="https://busybox.net/downloads/BusyBox.html#commands">busybox</a></li>
</li>

notes:
- fewer code paths, less of the specific glibc legacy, but more edge cases (locales, glibc specifics)
- busybox: single binary version 300 linux tools - coreutils replace
- together: alpine; 5MB linux distro
- don't pick scratch, 5MB small price to pay for bash and a working package manager
- don't pick ubuntu unless you need to (order of magnitude more disk space)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
alpine based image

```dockerfile
FROM alpine:3.9
RUN apk --no-cache add ca-certificates && \
    adduser usr -Du 1000 -h /app
COPY --chown=usr:usr ./controller /app/
EXPOSE 8080
USER usr
WORKDIR /app
ENTRYPOINT ["/app/controller"]
```

notes:
- always pin your deps (otherwise why even lockfile)
- dont apk upgrade (3.9 tag updates should receive security upgrades)
- ca-certs if you need tls (if you have a service mesh, you might not)
- rest is just like scratch, copy prebuilt static binary

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
compiling for musl (pure rust)

```sh
rustup target add x86_64-unknown-linux-musl
cargo build --target=x86_64-unknown-linux-musl --release
```

notes:
- easiest locally, might work nice on travis
- but C deps..
- openssl big one (tho might be replaced by rustls.. benches)
- psql (or other sql client (requires --with-openssl build)
- curl (but not needed anymore due to hyper)
- zlib (serving anything gzipped, but miniz.h works)
- other library that has C bindings (that's not self-contained in -sys crate)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
compiling for musl (with C dependencies)

```sh
docker run --rm \
  -v $PWD:/volume \
  -t clux/muslrust:stable \
  cargo build --release
```

notes:
- curl, openssl, psql, zlib, sqlite
- one of 3-4 big musl build images (this is the only one that builds continually and makes descriptive tags), 2M dls
- push like 20GB a month from travis for free for this
- bump stable every 6w, and script to auto-bump deps
- my image, so you may want to fork it in a company.. really rust should support it (but no feedback on that)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
compiling for musl (with C dependencies)

```sh
docker run --rm \
  -v $PWD:/volume \
  -v cargo-cache:/root/.cargo/registry \
  -t clux/muslrust:stable \
  cargo build --release
```
notes:
- how to add a cache
- can also do .cargo/git

---
ci flow

```
1. cargo test
1. cargo build --release
2. docker build && docker push
2. github releases
```

notes:
- 1 stashes musl binary, 2 attaches it from prev (so super quick)
- 2 master only (maybe tag only)
- releasing for cli bins (may do multiple release builds)
- what is multistage except removing that possibility for a tiny encapulation win?

---
caching (circleci)

```yaml
  musl_build:
    docker: [ { image: clux/muslrust:stable-1.36.0 } ]
    steps:
      - checkout
      - restore_cache: { keys: ["registry"] }
      - restore_cache: { keys: ["target"] }
      - run: cargo build --release
```
```yaml
      - save_cache:
          key: target
          paths: ["target"]
      - save_cache:
          key: registry
          paths: ["/root/.cargo"]
      - run: mv target/MUSLTRIPLE/release/app{,.MUSLTRIPLE}
      - persist_to_workspace:
          root: target/MUSLTRIPLE/release/
          paths: ["app.MUSLTRIPLE"]
```
[cargo#5026](https://github.com/rust-lang/cargo/issues/5026)

notes:
- minimal circle steps with caching
- caches two things: registry (DOWNLOADING CRATE), target (COMPILING CRATE)
- {TRIPLE} = `x86_64-unknown-linux-musl`
- additive unbounded; add evar, cargo clean, or base cache on sha(lockfile)
- sha(lockfile) is inefficient depending on release cadence
- cargo clean bug, cargo sweep

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
- abbreviating awkward circle yaml 4general idea
- PR build from branch: (cargo test + musl build)
- master merge: (musl build + docker build)
- github release: (musl + mac build then docker build + github_release)
- test + build in musl? may as well.

---
<!-- .slide: data-background-image="./zoid-resignation.webp" data-background-size="100% auto" class="color"-->

notes:
- lezz get our rust integrated with k8s

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Kubernetes Deployment

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

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
kube crate

```toml
kube = "0.11.0"
```

```toml
kube = { version = "0.11.0", features = ["openapi"] }
k8s-openapi = { version = "0.4.0", features = ["v1_13"] }
```

notes:
- kube crate (there's 3, this is _the one_ that tries to do a simplified api and higher level concepts)
- either plain, great for just crds or no native objs (or writing self)
- openapi structs have all the things but pretty heavy dep
- if you wanna write a subset of structs (memory, or less deps) you can..



---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Using kube

```rust
let cfg = kube::config::incluster_config().or_else(|_| {
    kube::config::load_kube_config()
}).expect("Failed to load kube config");
let client = kube::client::APIClient::new(cfg)?;
```

<p class="fragment">..[..impersonate locally](https://clux.github.io/probes/post/2019-03-31-impersonating-kube-accounts/)</p>
TODO: merge eks

notes:
- LOADING CONFIG: auto injected in-cluster SA, or config
- local dev might not work if you have auth hooks or oidc providers (can impersonate)
- BUT ONCE YOU HAVE A CLIENT, YOU CAN

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Using the kube api

```rust
let pods = Api::v1Pod(client).within("default");
let blog = pods.get("blog")?;
println!("blog has containers: {:?}", blog.spec.containers);

for p in pods.list(&ListParams::default())?.items {
    println!("found pod", p.name);
    pods.delete(p.name)?;
}
```

notes:
- create one of the variants of the generic Api object
- can do all the things
- mirrors client-go (but 20x smaller, 80k vs 5k)
- 5 lines per native obj, group/api/version + struct maps

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Using the kube api

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
- quick spec with serde json!
- attach objs to parts of it

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Under the hood

```rust
impl<K> Api<K> where
    K: Clone + DeserializeOwned + KubeObject
{
  fn get(&self, name: &str) -> Result<K>;
  fn create(&self, data: Vec<u8>) -> Result<K>;
  fn patch(&self, name: &str, patch: Vec<u8>) -> Result<K>;
  fn replace(&self, name: &str, data: Vec<u8>) -> Result<K>;
    ```
  ```rust
  fn list(&self) -> Result<ObjectList<K>>;
  fn watch(&self, version: &str)
    -> Result<Vec<WatchEvent<P, U>>>;
  fn delete_collection(&self)
    -> Result<Either<ObjectList<K>, Status>>;
  fn delete(&self, name: &str)
    -> Result<Either<K, Status>>;
}
```

notes:
- can treat each KubeObject generically - following kube apimachinery assumptions
- slightly simplified signatures (Params object for all but get)
- hidden subresources like get_scale, replace_status, done most core objs
- delete either (like 201 created)
- some special verbs missing (drain on nodes, logs on pods)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Using patch

```rust
let patch = json!({
    "spec": { "info": "patched baz" }
});
let o = foos.patch("baz", &pp, serde_json::to_vec(&patch)?)?;
assert_eq!(o.spec.info, "patched baz");
assert_eq!(o.spec.name, "baz");
```

notes:
- patching great for partially modifying an object
- patching in controlles for separating concerns
- controllers can own different part of .status (one writer for each field)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

why write a controller?

<ul>
<li class="fragment">separation of concerns</li>
<li class="fragment">distributed, self-healing systems</li>
<li class="fragment"><em>not writing a monolith</em></li>
</ul>
notes:
- one resource deals with one type of event (beats cm monorepo that grow and intermingle, a controller CANT reach out by RBAC)
- reconciliation loops running constantly can help you respond to errors in a system faster than 
- why you? you should have an answer to this yourself.
- if no answer? well, hope you enjoy the rust...

- old model: having N CI jobs per cluster, triggering them periodically, implementing diffing (do i need to do something) yourself, reconcile state in the job
- new model: write one tiny app, in a way that's meant to deal with events, deploy it to all clusters

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Controller scope

<ul>
  <li class="fragment">[Kubernetes Control Plane for Busy People Who Like Pictures](https://www.youtube.com/watch?v=zCXiXKMqnuE)</li>
  <li class="fragment">[Writing Kubernetes Controllers for CRDs](https://www.youtube.com/watch?v=7wdUa4Ulwxg)</li>
  <li class="fragment">[Writing Kube Controllers for Everyone](https://www.youtube.com/watch?v=AUNPLQVxvmw)</li>
  <li class="fragment">[CNCF youtube](https://www.youtube.com/channel/UCvqbFHwN-nwalWPjPUKpvTA)</li>
</li>

notes:
- limiting my talk to rust stuff, assuming some CRD familiarity here.. BUT:
- busy people: all controllers, great 4 learning kube + ctrl scope KCon2019
- Writing Controllers for CRDs (doing it in go, shows what a ballache it is)
- Last one, high level. Useful for knowing why controllers are what they are.
- CNCF channel has all kubecon videos

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Controller patterns

```basic
10 WATCH CustomResource:
     WHILE WatchEvent
         RECONCILE(CustomResource, event)
20 GOTO 10
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
  <li class="fragment">`Informer<K>` (event informer)</li>
  <li class="fragment">`Reflector<K>` (in-memory cache)</li>
</li>

notes:
- Informer: lets you handle raw events as they come in
- Use case: make changes to it/other resources when events occur (controller pattern)
- Reflector: maintains an in memory cache of all Ks
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
- error watch event: bookmark

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

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: foos.clux.dev
spec:
  group: clux.dev
  names:
    kind: Foo
    listKind: FooList
    plural: foos
    singular: foo
  scope: Namespaced
```
```yaml
  version: v1
  versions:
  - name: v1
    served: true
    storage: true
  subresources:
    status: {}
```
notes:
- pass some initial yaml to kube api server, ~20 lines generally
- formality; can be lax, can define openapi schema.

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
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
let inf = Informer::new(foos)
    .timeout(15)
    .init()?;
```
notes:
- can make informers + reflectors on them
- from there you can basically just start handling events
- and ensuring that you can reconcile changes
- but something needs to drive it...

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Driving an Informer

```rust
loop {
    inf.poll()?;
    while let Some(event) = inf.pop() {
        reconcile(&client, event)?;
    }
}
```

notes:
- rwlock internally, but otherwishe shareable informer/reflector
- poll writes to the state, pop extracts
- similar for reflector
- loop to drive in ex, if you have other stuff (web), make a thread

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Putting it together

```rust
#[derive(Clone)]
pub struct Controller {
    inf: Informer<Foo>,
    state: Arc<RwLock<State>>,
    metrics: Arc<RwLock<Metrics>>,
    client: APIClient,
}
```

```rust
#[derive(Clone)]
pub struct Watcher {
    rf: Reflector<Deploy>,
    state: Arc<RwLock<Vec<Entry>>>,
    client: APIClient,
}
```

notes:
- object to share with actix state
- interface and metrics etc, business logic

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Putting it together - main

```rust
fn main() {
  let cfg = kube::config::incluster_config().or_else(|_| {
      kube::config::load_kube_config()
  }).expect("Load kube config");
  let watcher = Watcher::init(cfg).expect("Watcher init");
```
```rust
  let sys = actix::System::new("watcher");
  HttpServer::new(move || {
    App::new()
      .data(watcher.clone())
      .wrap(middleware::Logger::default().exclude("/health"))
      .service(web::resource("/").to(index))
      .service(web::resource("/health").to(health))
    })
    .bind("0.0.0.0:8000").expect("Bind 0.0.0.0:8000")
    .shutdown_timeout(0)
    .start();
  let _ = sys.run();
}
```

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Putting it together - routes

```rust
fn index(w: Data<Watcher>, _: HttpRequest) -> HttpResponse {
    HttpResponse::Ok().json(w.state())
}
fn health(_: HttpRequest) -> HttpResponse {
    HttpResponse::Ok().json("healthy")
}
```

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Putting it together - extra crates

<ul>
  <li class="fragment">prometheus / actix_web_prom</li>
  <li class="fragment">env_logger</li>
  <li class="fragment">sentry / sentry_actix <small>([sentry-rust#143](https://github.com/getsentry/sentry-rust/issues/143))</small></li>
</li>

notes:
- custom metrics => prometheus library, actix transform for no mesh
- actix_web_prom does basic latency metrics in two lines (.wrap)
- env_logger (precise RUST_LOG drilling really useful, tokio spammy)
- sentry actix integration, 2 lines, panic handler + middleware (unf. broken in actix 1.0)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Examples

<ul>
  <li class="fragment">[kube-rs/examples](https://github.com/clux/kube-rs/tree/master/examples)</li>
  <li class="fragment">[version-rs](https://github.com/clux/version-rs) (super light, ~100 lines)</li>
  <li class="fragment">[controllers-rs](https://github.com/clux/controller-rs) (circleci, kube yaml, 3 file split)</li>
</ul>

notes:
- kube ex for reflectors, informer, basics + api usage
- controller-rs for a 3 file actix setup with informers, prometheus
- version-rs for a 100 line watcher with actix and metrics, actix prom

---
<!-- .slide: data-background-image="./zoid-no.webp" data-background-size="100% auto" class="color"-->

notes:
- battered

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

- Eirik - [github.com/clux](https://github.com/clux) - [clux.dev](https://clux.dev)
- Babylon Health - [babylonhealth.com](https://babylonhealth.com)

<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 560px" />
