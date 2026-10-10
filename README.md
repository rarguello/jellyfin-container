# Jellyfin on OpenShift

Run [Jellyfin](https://jellyfin.org) on OpenShift and stream your library
**directly from [pCloud](https://www.pcloud.com/)** through an
[rclone](https://rclone.org) FUSE sidecar — no local copy of the media.

Everything is plain manifests, applied with Kustomize. No Helm, no operators.

## How it works

A single Pod runs two containers:

- **jellyfin** — the media server, listening on `:8096`.
- **rclone** — mounts the pCloud remote as a read-only FUSE filesystem.

The `rclone` container mounts pCloud at `/mnt/shared/pcloud` inside a shared
`emptyDir` and propagates that mount to the Jellyfin container with
`mountPropagation: Bidirectional` / `HostToContainer`. In Jellyfin the same tree
appears at `/media/pcloud`, and the library points at `/media/pcloud/<folder>`.

FUSE needs `/dev/fuse` and `CAP_SYS_ADMIN`, so the Pod runs under a dedicated
`SecurityContextConstraints` (`jellyfin-fuse`) bound to the `jellyfin`
ServiceAccount. That SCC takes `fsGroup` and `runAsUser` from the namespace's
assigned range, so the manifests carry no cluster-specific IDs.

```
Pod
├─ init: mkdir /mnt/shared/pcloud
├─ rclone  ── mounts pcloud: ─► /mnt/shared/pcloud   (privileged)
└─ jellyfin ◄── mountPropagation ─ /media/pcloud     (reads the FUSE mount)
```

## Requirements

- OpenShift 4.x.
- A pCloud account and an rclone remote named `pcloud` (create it with
  `rclone config`).
- Cluster-admin (or equivalent) to create the SCC.

## Deploy

```sh
# 1. rclone config (gitignored; fed to the secretGenerator)
mkdir -p base/secrets
cp ~/.config/rclone/rclone.conf base/secrets/

# 2. apply the base
oc apply -k base
```

For cluster-specific values (e.g. the published URL) keep a local, gitignored
overlay in `overlays/live/` and apply that instead:

```sh
oc apply -k overlays/live
```

Then open the Route and finish Jellyfin's setup wizard.

## Configuration

- **Timezone** — the base sets `TZ=UTC`; override it in your overlay (or in
  `base/deployment.yaml`).
- **Published server URL** — set `JELLYFIN_PublishedServerUrl` in
  `deployment.yaml` (defaults to the placeholder `https://jellyfin.example.com`),
  or set it in Jellyfin under *Dashboard → Networking*.
- **Library path** — add a library pointing at `/media/pcloud/<your-folder>`.
- **Storage** — `base/pvc.yaml` uses the cluster's default StorageClass; add
  `storageClassName` if you need a specific one.
- **Namespace** — the SCC's `users` field hardcodes
  `system:serviceaccount:jellyfin:jellyfin`. If you change the namespace, update it.
- **rclone cache** — `--vfs-cache-max-size` (30G) must stay well below the
  `rclone-cache` emptyDir `sizeLimit` (40Gi). rclone can briefly overshoot its
  cache limit, and the kubelet evicts the Pod if the emptyDir fills up.

## Known limitations

- **In-place container restarts get stuck.** If the `jellyfin` container restarts
  while the FUSE mount is present, the kubelet's recursive SELinux relabel fails on
  the read-only FUSE mount (`relabel failed ... read-only file system`) and the Pod
  never comes back. Delete the Pod to recover (`oc delete pod -l app=jellyfin`).
  A node reboot is fine — the Pod is recreated with a fresh, empty shared volume.

## Files

| File | Purpose |
|---|---|
| `base/kustomization.yaml` | resources + `secretGenerator` for the rclone config |
| `base/namespace.yaml` | the `jellyfin` Namespace |
| `base/scc.yaml` | `jellyfin-fuse` SCC (privileged for FUSE) |
| `base/rbac.yaml` | the `jellyfin` ServiceAccount |
| `base/pvc.yaml` | `jellyfin-config` PVC (5Gi) |
| `base/deployment.yaml` | Jellyfin + rclone sidecar |
| `base/service.yaml` | ClusterIP Service on `:8096` |
| `base/route.yaml` | edge-TLS Route |
| `base/secrets/rclone.conf` | **gitignored** — your pCloud token |
| `overlays/live/` | **gitignored** — cluster-specific overlay |

## Gotchas

- Disable chapter-image and trickplay generation in Jellyfin **before** adding the
  library. Otherwise Jellyfin runs ffmpeg over every file to build thumbnails, and
  rclone downloads the whole library.
- The rclone mountpoint must be a subdirectory of the shared volume; the volume
  root is already a mount point.
- Transcoding reads the whole file through rclone; direct play streams on demand.
