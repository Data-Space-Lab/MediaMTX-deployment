# MediaMTX deployment

This repository deploys a MediaMTX relay for the DIL connector RTSP dataplane.
The MediaMTX control API is exposed only inside the `dil-connector` namespace.
The RTSP service is a separate NodePort service for raw RTSP clients. A small
same-pod HTTP player also exposes MediaMTX HLS playback for browsers.

## Deploy with ArgoCD

Apply `argocd-application.yaml` to the material vcluster's `argocd` namespace.
The application deploys MediaMTX into the existing `dil-connector` namespace.

The application uses the pinned upstream image `bluenviron/mediamtx:v1.21.1`.
MediaMTX is configured with a 128 MiB memory request and a 256 MiB memory
limit because each active RTSP relay and HLS reader consumes memory. If the
pod is restarted, its dynamically-created transfer paths are lost and active
transfers must be started again.

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

## Browser player

Browsers cannot play an `rtsp://` URL directly. The deployment enables
MediaMTX HLS and serves a small player at `/player/`. Route the player service
through an HTTP/HTTPS gateway, then open:

```text
https://<player-host>/player/?path=<transfer-session-path>
```

The `path` value is the opaque session path returned by the RTSP dataplane,
not the full RTSP URL. The player proxies HLS through `/hls/` and does not
expose the MediaMTX control API. Safari uses native HLS; other supported
browsers use hls.js.

## Services

| Service | Port | Purpose |
| --- | ---: | --- |
| `dil-connector-mediamtx` | `9997` | Private MediaMTX control API |
| `dil-connector-mediamtx-rtsp` | `8554` / NodePort `30554` | RTSP relay traffic |
| `dil-connector-mediamtx` | `8080` | Browser player and HLS proxy |

The deployment does not expose the MediaMTX control API externally. The RTSP
service is exposed as a dedicated TCP NodePort because the shared Envoy Gateway
currently has HTTP/HTTPS listeners only; an `HTTPRoute` cannot carry RTSP.

## Health checks

MediaMTX's v3 control API is used for readiness and liveness checks. The
dataplane's Verify action should report that the MediaMTX control API is
reachable before a transfer is started.
