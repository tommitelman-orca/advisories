# OpenKruise kruise-manager: unauthenticated pprof enabled by default allows cluster-wide admission outage

| | |
|---|---|
| **Product** | OpenKruise ([openkruise/kruise](https://github.com/openkruise/kruise)), kruise-manager |
| **Affected** | up to and including v1.9.1 |
| **Fixed** | [v1.9.2](https://github.com/openkruise/kruise/releases/tag/v1.9.2) (2026-10-19) |
| **Author** | Tom Mitelman, Orca Security |
| **Date** | 2026-11-02 |
| **CVE ID** | [CVE-2026-120656](https://www.cve.org/CVERecord?id=CVE-2026-120656) (CERT/CC VU#742684) |
| **Severity** | High: any pod on the cluster network can take down the OpenKruise admission webhooks and block pod creation cluster-wide, with no credentials |
| **CWE** | [CWE-306](https://cwe.mitre.org/data/definitions/306.html) Missing Authentication for Critical Function (root cause), [CWE-400](https://cwe.mitre.org/data/definitions/400.html) Uncontrolled Resource Consumption (impact) |

## 1. Summary

Up to v1.9.1, kruise-manager starts Go `net/http/pprof` by default on `:8090`, which binds all interfaces, with no authentication. kruise-manager also serves the OpenKruise admission webhooks. An unauthenticated caller can exhaust the manager's memory through pprof until it goes NotReady, and because the webhooks fail closed, pod and workload creation then fails across the cluster.

## 2. Details

In [`main.go` at v1.9.1](https://github.com/openkruise/kruise/blob/v1.9.1/main.go):

- [L23](https://github.com/openkruise/kruise/blob/v1.9.1/main.go#L23): `net/http/pprof` is imported, which registers its handlers on `http.DefaultServeMux`
- [L118-L119](https://github.com/openkruise/kruise/blob/v1.9.1/main.go#L118-L119): `--enable-pprof` defaults to `true` and `--pprof-addr` defaults to `:8090`
- [L156](https://github.com/openkruise/kruise/blob/v1.9.1/main.go#L156): `http.ListenAndServe(pprofAddr, nil)` serves that mux with no authentication

The Helm chart sets `manager.pprofAddr: "localhost:8090"`, but that value is not passed to the manager container, so the all-interfaces default applies in chart installs too. Metrics (`:8080`) and health (`:8000`) are separate listeners and are not affected.

## 3. Impact

From any pod that can reach the kruise-manager pod IP on port 8090, with no Kubernetes RBAC:

- Read the manager's goroutine stacks and heap profiles
- Denial of service. In my test, repeated expensive pprof requests grew the manager from about 84 to 329 MiB until both replicas went NotReady. The OpenKruise admission webhooks use `failurePolicy: Fail`, so while the manager was down, pod creation failed in an unrelated namespace (`kubectl run` returned connection refused from the webhook). The manager recovered when the requests stopped.

## 4. Remediation

- Upgrade to v1.9.2 or later. [PR #2613](https://github.com/openkruise/kruise/pull/2613) changes the `--enable-pprof` default to `false` in kruise-manager and kruise-daemon
- If you cannot upgrade yet, add `--enable-pprof=false` to the kruise-manager container args (the chart has no value for it), or restrict ingress to the kruise-manager pods on port 8090 with a NetworkPolicy

## 5. Disclosure timeline

- 2026-09-27: reported to CERT/CC (VRF#26-09-QKKTS)
- 2026-10-11: reported to the OpenKruise maintainers
- 2026-10-14: fix merged (PR #2613)
- 2026-10-19: v1.9.2 released
- 2026-11-02: CERT/CC case VU#742684 opened for CVE assignment
- 2026-11-02: this advisory published
- 2026-11-02: CVE-2026-120656 assigned by CERT/CC

## 6. References

- Fix: https://github.com/openkruise/kruise/pull/2613
- Release: https://github.com/openkruise/kruise/releases/tag/v1.9.2
- Affected code: https://github.com/openkruise/kruise/blob/v1.9.1/main.go#L118-L119
