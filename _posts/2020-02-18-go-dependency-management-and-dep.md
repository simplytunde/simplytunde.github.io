---
layout: post

---

You may be aware of the "official experiment" for go dependency management tool called **dep** but this has since be replaced by **go mod**. Let's explore the tool via this opensource [project](https://github.com/knative/serving/blob/master/Gopkg.toml) usage.  `Gopkg.toml` is where you define your package dependencies while `Gopkg.lock` contains snapshot of your project dependencies after evaluating `Gopkg.toml` as well as some metadata. Here is;

```go
# Refer to https://github.com/golang/dep/blob/master/docs/Gopkg.toml.md
# for detailed Gopkg.toml documentation.

required = [
  ...
  "knative.dev/pkg/codegen/cmd/injection-gen",
  # TODO(#4549): Drop this when we drop our patches.
  "k8s.io/kubernetes/pkg/version",
  "knative.dev/caching/pkg/apis/caching",
  # For cluster management in performance testing.
  "knative.dev/pkg/testutils/clustermanager/perf-tests",
  "knative.dev/test-infra/scripts",
  "knative.dev/test-infra/tools/dep-collector",
  # For load testing.
  "github.com/tsenart/vegeta"
]

[[constraint]]
  name = "github.com/tsenart/vegeta"
  branch = "master"

[[override]]
  name = "gopkg.in/yaml.v2"
  version = "v2.2.4"
...
[[override]]
  name = "github.com/google/mako"
  version = "v0.1.0"

[[override]]
  name = "go.uber.org/zap"
  revision = "67bc79d13d155c02fd008f721863ff8cc5f30659"
...
[[constraint]]
  name = "github.com/jetstack/cert-manager"
  version = "v0.12.0"
...
[[override]]
  name = "k8s.io/api"
  version = "kubernetes-1.16.4"
...
[[override]]
  name = "k8s.io/kube-openapi"
  # This is the version at which k8s.io/apiserver depends on this at its 1.16.4 tag.
  revision = "743ec37842bffe49dd4221d9026f30fb1d5adbc4"
...

# Added for the custom-metrics-apiserver specifically
[[override]]
  name = "github.com/kubernetes-incubator/custom-metrics-apiserver"
  revision = "3d9be26a50eb64531fc40eb31a5f3e6720956dc6"

[[override]]
  name = "bitbucket.org/ww/goautoneg"
  source = "github.com/munnerz/goautoneg"

[prune]
  go-tests = true
  unused-packages = true
  non-go = true

[[prune.project]]
  name = "k8s.io/code-generator"
  unused-packages = false
  non-go = false

[[prune.project]]
  name = "knative.dev/test-infra"
  non-go = false

...

# The dependencies below are required for opencensus.
[[override]]
  name = "google.golang.org/genproto"
  revision = "357c62f0e4bbba7e6cc403ae09edcf3e2b9028fe"

[[override]]
  name = "contrib.go.opencensus.io/exporter/prometheus"
  version = "0.1.0"

[[override]]
  name = "contrib.go.opencensus.io/exporter/zipkin"
  version = "0.1.1"

[[constraint]]
  name = "go.opencensus.io"
  version = "0.22.0"

[[override]]
  name = "github.com/census-instrumentation/opencensus-proto"
  version = "0.2.0"

[[override]]
  name="github.com/golang/protobuf"
  version = "1.3.2"

```

#### required

Lets talk about this required section; this is where you define what dependencies are required and must be included in the vendor folder;

```bash
required = [
  ...
  "knative.dev/pkg/codegen/cmd/injection-gen",
  # TODO(#4549): Drop this when we drop our patches.
  "k8s.io/kubernetes/pkg/version",
  "knative.dev/caching/pkg/apis/caching",
  # For cluster management in performance testing.
  "knative.dev/pkg/testutils/clustermanager/perf-tests",
  "knative.dev/test-infra/scripts",
  "knative.dev/test-infra/tools/dep-collector",
  # For load testing.
  "github.com/tsenart/vegeta"
]
```

If you look at the folder, you will see these dependencies in the [folder](https://github.com/knative/serving/tree/master/vendor) as these are required. 

#### direct and transitive dependency

`A -> B -> C -> D`

These are packages that your project imports or includes the required sections. For example above, A directly import B while B imports C and D. In this case, B is the direct dependent of A while C and D are transitive dependency.

#### constraint

This is how you specify the version of your dependency to use for this project. In our case, we want  our project to use version`v2.2.4` of the `gopkg.in/yaml.v2`. To do this,

```
[[override]]
  name = "gopkg.in/yaml.v2"
  version = "v2.2.4"
```

You can also use `branch` or `revision` to pin your dependency.

#### override

These are like global constraint but supersedes constraints and should be used as last resort. They apply to direct dependencies and transitive dependencies unlike constraint and advisably should be used sparingly.

```
[[override]]
  name="github.com/golang/protobuf"
  version = "1.3.2"
```



#### prune

When your project has a dependency, it extracts the package along with other files like README.md, LEGAL etc. If you dont want any or all of these files, you can use prune to inform dep.

```
[prune]
  go-tests = true
  unused-packages = true
  non-go = true

[[prune.project]]
  name = "k8s.io/code-generator"
  unused-packages = false
  non-go = false

[[prune.project]]
  name = "knative.dev/test-infra"
  non-go = false
```

The setting above simply tells dep to remove test files, unused-packages as well as non-go files like LEGAL from all dependencies. This is the overriden with a different setting for `k8s.io/code-generator`   that unused-packages and no-go should not be pruned.



#### dep ensure

Once you have configured your Gopkg.toml with your settings, you need to apply these changes, update vendor folder as well as Gopkg.lock. `dep ensure` will do that for you and you have option to tell dep not to update  vendor, just Gopkg.lock.