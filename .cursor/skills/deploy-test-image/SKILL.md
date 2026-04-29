---
name: deploy-test-image
description: >-
  Build, push, and deploy a custom hypershift-addon-operator image to an MCE/ACM
  test environment using MCE image-override configmaps. Use when the user wants
  to test a code change on a live cluster, deploy a custom image, override the
  addon image, or revert a test deployment.
---

# Deploy Test Image to MCE/ACM Environment

## Prerequisites

- Docker with `buildx` support (for cross-platform builds)
- `oc` CLI authenticated to both ACM hub and MCE spoke clusters
- A public container registry (e.g. `quay.io/<user>/hypershift-addon-operator`)
- The MCE resource name (find with `oc get mce`)

## Method: MCE Image-Override Configmap

This is the **preferred** method. It uses the MCE installer's built-in
image-override annotation so the addon manager reconciles the custom image
itself — no need to scale down managers or fight image reverts.

### Step 1 — Build and Push

Build for `linux/amd64` (OpenShift nodes) and push directly:

```bash
docker buildx build --platform linux/amd64 \
  -t quay.io/<user>/hypershift-addon-operator:<tag> \
  --push .
```

> If the user has an Apple Silicon Mac, `--platform linux/amd64` is required.
> Using `--push` sends the image straight to the registry without loading it
> into the local daemon (which would fail on cross-platform builds).

Verify the image is public or the cluster can pull it:

```bash
docker manifest inspect quay.io/<user>/hypershift-addon-operator:<tag>
```

### Step 2 — Create the Image-Override Configmap (MCE spoke)

Log in to the **MCE spoke** cluster and create a configmap in the
`multicluster-engine` namespace. The `image-key` must be
`hypershift_addon_operator`:

```bash
oc login <MCE_API_URL> -u kubeadmin -p <password> --insecure-skip-tls-verify

cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: hypershift-addon-test-image
  namespace: multicluster-engine
data:
  manifest.json: |-
    [
      {
        "image-name": "hypershift-addon-operator",
        "image-remote": "quay.io/<user>",
        "image-tag": "<tag>",
        "image-key": "hypershift_addon_operator"
      }
    ]
EOF
```

### Step 3 — Annotate the MCE Resource

```bash
MCE_NAME=$(oc get mce -o jsonpath='{.items[0].metadata.name}')
oc annotate mce "$MCE_NAME" --overwrite \
  installer.multicluster.openshift.io/image-overrides-configmap=hypershift-addon-test-image
```

The MCE installer will roll out the new image to all addon-agent deployments
managed by the MCE. No manual scaling is needed.

### Step 4 — Verify Rollout

```bash
# Check the agent image in the discovery namespace
oc get deployment hypershift-addon-agent \
  -n open-cluster-management-agent-addon-discovery \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="hypershift-addon-agent")].image}'

# Watch rollout
oc rollout status deployment/hypershift-addon-agent \
  -n open-cluster-management-agent-addon-discovery

# Tail logs
oc logs -f deployment/hypershift-addon-agent \
  -n open-cluster-management-agent-addon-discovery \
  -c hypershift-addon-agent
```

## Revert to Original Image

Remove the annotation and delete the configmap:

```bash
MCE_NAME=$(oc get mce -o jsonpath='{.items[0].metadata.name}')
oc annotate mce "$MCE_NAME" \
  installer.multicluster.openshift.io/image-overrides-configmap- --overwrite
oc delete configmap hypershift-addon-test-image -n multicluster-engine --ignore-not-found
```

The MCE installer will reconcile the agent back to its original image.

## Quick Reference

| Action | Command |
|--------|---------|
| Build + push | `docker buildx build --platform linux/amd64 -t quay.io/<user>/hypershift-addon-operator:<tag> --push .` |
| Apply override | `oc annotate mce $MCE_NAME --overwrite installer.multicluster.openshift.io/image-overrides-configmap=hypershift-addon-test-image` |
| Remove override | `oc annotate mce $MCE_NAME installer.multicluster.openshift.io/image-overrides-configmap- --overwrite` |
| Check image | `oc get deploy hypershift-addon-agent -n open-cluster-management-agent-addon-discovery -o jsonpath='{..image}'` |

## Important Notes

- The `image-key` must be `hypershift_addon_operator` — this is the key the
  MCE installer uses internally to resolve the addon image.
- If overriding on an **ACM hub** (two-cluster setup), apply the same configmap
  and annotation on the ACM hub's MCE resource as well, if you need the hub's
  copy of the addon to also use the custom image.
- Always revert overrides when testing is complete to avoid stale images.
- This method replaces the old approach of `oc set image` + scaling down
  managers, which was fragile and required fighting reconciliation loops.
