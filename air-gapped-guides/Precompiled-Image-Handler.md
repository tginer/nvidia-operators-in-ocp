# NIC Cluster Policy driver image pulling
_Exported on 12/16/2025 at 10:08:32 GMT+1 from Cursor (2.1.49)_

## Summary: Who Handles `forcePrecompiled: true`

When `forcePrecompiled: true` is set in the NIC Cluster Policy, there are **two components involved** in different phases:

### 1. **Network Operator Controller Pod** (Checks if precompiled image exists)

**Pod**: `nvidia-network-operator-*` (runs in the `nvidia-network-operator` namespace)  
**Container**: The main operator container

The Network Operator controller has a `docadriverimages.Provider` component that:

```51:63:pkg/docadriverimages/doca_drivers.go
// NewProvider creates a provider for  DOCA driver images,
// queries the container image registry to get the exiting tags
func NewProvider(ctx context.Context, c client.Client) Provider {
	p := &provider{c: c, docaImageSpec: nil, tags: make([]string, 0), ctx: ctx}

	ticker := time.NewTicker(time.Duration(config.FromEnv().State.DocaDriverImagePollTimeMinutes) * time.Minute)

	go func() {
		for ; ; <-ticker.C {
			p.retrieveTags()
		}
	}()
	return p
}
```

This provider **periodically queries the container registry** (e.g., `nvcr.io/nvidia/mellanox/doca-driver`) to get all available image tags.

When reconciling, it checks if the precompiled tag exists:

```548:554:pkg/state/state_ofed.go
	precompiledTag := fmt.Sprintf(precompiledTagFormat, cr.Spec.OFEDDriver.Version, nodePool.Kernel,
		nodePool.OsName, nodePool.OsVersion, nodePool.Arch)
	precompiledExists := docaProvider.TagExists(precompiledTag)
	reqLogger.V(consts.LogLevelDebug).Info("Precompiled tag", "tag:", precompiledTag, "found:", precompiledExists)
	if !precompiledExists && cr.Spec.OFEDDriver.ForcePrecompiled {
		return nil, fmt.Errorf("ForcePrecompiled is enabled and precompiled tag was not found: %s", precompiledTag)
	}
```

If `forcePrecompiled: true` and the precompiled tag **does NOT exist** in the registry, the reconciliation **fails with an error** and no DaemonSet is created.

### 2. **MOFED DaemonSet Pods** (Actually pulls and runs the driver image)

**Pod**: `mofed-<osname><osver>-<kernelhash>-ds-*` (e.g., `mofed-ubuntu22.04-fgxyz-ds-abc123`)  
**Container**: `mofed-container`

If the precompiled tag exists, the Network Operator generates the MOFED DaemonSet with the precompiled image name:

```668:672:pkg/state/state_ofed.go
	if precompiledExists {
		return fmt.Sprintf(precompiledImageFormat,
			cr.Spec.OFEDDriver.Repository, cr.Spec.OFEDDriver.Image,
			cr.Spec.OFEDDriver.Version, pool.Kernel,
			pool.OsName, pool.OsVersion, pool.Arch)
```

The image name format for precompiled is:
```
<repo>/<image>:<version>-<kernel>-<os><osver>-<arch>
```
Example: `nvcr.io/nvidia/mellanox/doca-driver:5.7-0.1.2.0-0-5.15.0-91-generic-ubuntu22.04-amd64`

The **kubelet on each worker node** then pulls this image when the DaemonSet pods are scheduled.

---

### Visual Flow

```
┌─────────────────────────────────────────────────────────────────┐
│               Network Operator Controller Pod                    │
│  (nvidia-network-operator namespace)                            │
│                                                                 │
│   1. docadriverimages.Provider queries container registry       │
│   2. Checks if precompiled tag exists for each node pool        │
│   3. If forcePrecompiled=true && tag missing → ERROR            │
│   4. If tag exists → generates DaemonSet with precompiled image │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              MOFED DaemonSet (per node pool)                    │
│   mofed-<os><ver>-<kernelhash>-ds                               │
│                                                                 │
│   Container: mofed-container                                    │
│   Image: nvcr.io/.../doca-driver:<ver>-<kernel>-<os>-<arch>     │
│                                                                 │
│   kubelet on each worker node pulls this precompiled image      │
└─────────────────────────────────────────────────────────────────┘
```

<!-- ```go
// NewProvider creates a provider for  DOCA driver images,
// queries the container image registry to get the exiting tags
func NewProvider(ctx context.Context, c client.Client) Provider {
	p := &provider{c: c, docaImageSpec: nil, tags: make([]string, 0), ctx: ctx}

	ticker := time.NewTicker(time.Duration(config.FromEnv().State.DocaDriverImagePollTimeMinutes) * time.Minute)

	go func() {
		for ; ; <-ticker.C {
			p.retrieveTags()
		}
	}()
	return p
}
```

```go
	precompiledTag := fmt.Sprintf(precompiledTagFormat, cr.Spec.OFEDDriver.Version, nodePool.Kernel,
		nodePool.OsName, nodePool.OsVersion, nodePool.Arch)
	precompiledExists := docaProvider.TagExists(precompiledTag)
	reqLogger.V(consts.LogLevelDebug).Info("Precompiled tag", "tag:", precompiledTag, "found:", precompiledExists)
	if !precompiledExists && cr.Spec.OFEDDriver.ForcePrecompiled {
		return nil, fmt.Errorf("ForcePrecompiled is enabled and precompiled tag was not found: %s", precompiledTag)
	}
```

```go
	if precompiledExists {
		return fmt.Sprintf(precompiledImageFormat,
			cr.Spec.OFEDDriver.Repository, cr.Spec.OFEDDriver.Image,
			cr.Spec.OFEDDriver.Version, pool.Kernel,
			pool.OsName, pool.OsVersion, pool.Arch)
```

```plaintext
<repo>/<image>:<version>-<kernel>-<os><osver>-<arch>
``` -->
### retrieveTags() function
retrieveTags() (lines 106-146)
```go
func (p *provider) retrieveTags() {
	if p.docaImageSpec == nil {
		return
	}
	p.mu.Lock()
	defer p.mu.Unlock()
	logger := log.FromContext(p.ctx)
	logger.Info("fetching DOCA driver image tags", "repo", p.docaImageSpec.Repository, "image", p.docaImageSpec.Image)
	pullSecrets := make([]corev1.Secret, 0)
	for _, name := range p.docaImageSpec.ImagePullSecrets {
		secret := &corev1.Secret{}
		err := p.c.Get(p.ctx, types.NamespacedName{
			Name:      name,
			Namespace: config.FromEnv().State.NetworkOperatorResourceNamespace,
		}, secret)
		if errors.IsNotFound(err) {
			continue
		} else if err != nil {
			logger.Error(err, "failed to get pull secret")
			return
		}
		pullSecrets = append(pullSecrets, *secret)
	}
	auth, err := kauth.NewFromPullSecrets(p.ctx, pullSecrets)
	if err != nil {
		logger.Error(err, "failed to create registry auth from secrets")
		return
	}
	image := fmt.Sprintf("%s/%s", p.docaImageSpec.Repository, p.docaImageSpec.Image)
	repo, err := name.NewRepository(image)
	if err != nil {
		logger.Error(err, "failed to create repo")
		return
	}
	tags, err := remote.List(repo, remote.WithAuthFromKeychain(auth))
	if err != nil {
		logger.Error(err, "failed to list tags")
		return
	}
	p.tags = tags
}
```

This function:
1. **Early exit** if `docaImageSpec` is nil (no OFED driver configured)
2. **Fetches image pull secrets** from the Kubernetes cluster (from the `nvidia-network-operator` namespace)
3. **Creates authentication** using [`go-containerregistry`](https://github.com/google/go-containerregistry) library's `kauth.NewFromPullSecrets()`
4. **Builds the repository path** (e.g., `nvcr.io/nvidia/mellanox/doca-driver`)
5. **Calls `remote.List()`** to query the container registry API and retrieve all available tags
6. **Caches the tags** in `p.tags` for subsequent `TagExists()` calls

This function is called:
- Periodically via a ticker (configured by `DocaDriverImagePollTimeMinutes`)
- Immediately when `SetImageSpec()` is called with a new/different image spec