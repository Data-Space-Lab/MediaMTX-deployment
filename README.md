# MediaMTX deployment

This repository deploys a MediaMTX relay for the DIL connector RTSP dataplane.
The MediaMTX control API is exposed only inside the `dil-connector` namespace.
The RTSP service is a separate ClusterIP service so it can be exposed through a
TCP/TLS-capable gateway when public playback is required.

## Deploy with ArgoCD

Apply `argocd-application.yaml` to the material vcluster's `argocd` namespace.
The application deploys MediaMTX into the existing `dil-connector` namespace.

The application uses the pinned upstream image `bluenviron/mediamtx:v1.21.1`.

## DIL dataplane configuration

Configure the material DIL dataplane RTSP adapter with:

```json
{
  "relayApiUrl": "http://dil-connector-mediamtx.dil-connector.svc.cluster.local:9997",
  "publicRtspBaseUrl": "rtsp://<public-stream-host>:8554",
  "sessionTtlSeconds": 3600,
  "requestTimeoutSeconds": 10
}
```

The private `relayApiUrl` is used by the dataplane to create and remove
per-transfer MediaMTX paths. It must never be advertised to consumers.

The `publicRtspBaseUrl` must resolve to an externally reachable TCP endpoint.
HTTP ingress routes cannot carry RTSP traffic. Expose the RTSP service through
a TCP-capable Gateway API route, load balancer, or equivalent network service.
The dataplane appends an opaque per-transfer path to this base URL.

## Services

| Service | Port | Purpose |
| --- | ---: | --- |
| `dil-connector-mediamtx` | `9997` | Private MediaMTX control API |
| `dil-connector-mediamtx-rtsp` | `8554` | RTSP relay traffic |

The deployment does not expose the MediaMTX control API externally.

## Health checks

MediaMTX's v3 control API is used for readiness and liveness checks. The
dataplane's Verify action should report that the MediaMTX control API is
reachable before a transfer is started.
