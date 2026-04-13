# Security / IAM Architecture Proposals

This repository contains GitHub-renderable architecture notes for discussing security, IAM, and token-handling options.

Each proposal lives in its own folder and uses Mermaid top-down flow diagrams inside `README.md` files so the content renders directly in GitHub.

The diagrams distinguish three zones where relevant:

- `Security Frontend`: WAF and API gateway as separate entry paths in front of the online cluster.
- `Online Cluster`: browser-facing IAM and backend workloads.
- `Intranet Cluster`: internal services and their own OIDC-compliant IAM.

How to read the diagrams:

- The numbered arrows are the main request path.
- `Cluster ingress` only routes traffic into the cluster. Validation happens in `oauth2-proxy`, the API gateway, the service, or mesh policy depending on the proposal.
- When a service calls `Online Keycloak`, that is the point where cluster identity can be used for workload identity. In the Keycloak 26.6.0-compatible variants, that means client authentication on the token request, not validation at ingress.
- The examples are illustrative protocol shapes, not full product configuration.
- The concrete example flow uses four business participants: `Browser UI`, `UI Service`, `Online Service`, and `Intranet Service`.

## Proposals

- [oauth2-proxy-no-client-token](./oauth2-proxy-no-client-token/README.md): `oauth2-proxy` integrated at cluster ingress, opaque browser session only, Keycloak token exchange for online APIs, and JWT bearer grant at the intranet IAM.
- [dpop-token-exchange](./dpop-token-exchange/README.md): Same basic shape as today, but the edge uses DPoP, online hops use Keycloak token exchange, and the intranet IAM exchanges trusted online tokens directly.
- [dpop-workload-identity](./dpop-workload-identity/README.md): Clear split between user identity and workload identity using DPoP at the edge, cluster identity for service-to-Keycloak authentication, Keycloak exchange for online hops, and JWT bearer grant plus optional workload auth at the intranet IAM.

## Comparison

| Proposal | Token in browser | User vs workload identity | Service-to-service model | Online IAM to intranet IAM trust |
| --- | --- | --- | --- | --- |
| Current state | Yes | Mostly coupled | Same user token is propagated | Implicit and provider-specific |
| `oauth2-proxy` edge session | No | Better separation at the edge | Internal APIs use exchanged tokens | Intranet IAM exchanges trusted online tokens directly |
| DPoP + token exchange | Yes, but sender-constrained on the edge | Separated on the online hops by exchanged tokens | No raw token chaining, each hop gets a new audience token | Intranet IAM exchanges trusted online tokens directly |
| Edge DPoP + workload identity | Edge token only | Strong separation | Workload identity authenticates the caller to Keycloak, exchanged tokens carry authorization context | Intranet IAM exchanges trusted online tokens directly |

The `Current state` row stays here only as a comparison baseline. It no longer has its own folder.

## Discussion Focus

- Whether the browser should ever hold an OAuth token at all.
- Whether DPoP is worth using inside the cluster or should stay an edge concern.
- How much authorization context should travel in exchanged access tokens versus workload identity.
- How explicit the online-token to intranet-IAM trust contract should be.

## Spec Notes

- `OIDC`: an identity layer on top of OAuth 2.0. In these proposals it is used for user authentication at the online IAM. The important outputs are the authenticated user session and the tokens issued after login.
- `DPoP`: a sender-constraining mechanism for OAuth tokens. The client sends a signed DPoP proof with the HTTP request, and the token can carry confirmation data that binds it to the proof key. In these proposals, DPoP protects tokens that would otherwise be replayable bearer tokens.
- `SPIFFE`: a standard identity format for workloads. A SPIFFE ID is a URI such as `spiffe://online.example/ns/apps/sa/ui-service`. In these proposals, SPIFFE is relevant only as a way to name the calling online workload.
- `WIMSE`: current IETF work on workload identity in multi-system environments. In these proposals, WIMSE is relevant as a direction for how workload identity and related security context can be conveyed to an IAM. It is not treated here as a finished wire protocol or a mandatory dependency.

## Validation Model

- `Cluster ingress` routes traffic into the online cluster.
- `API Gateway` may validate edge tokens or DPoP proofs if that is part of the proposal.
- `oauth2-proxy` may validate browser sessions where that is part of the proposal.
- `Service` or `Istio sidecar` may validate tokens inside the cluster where that is part of the proposal.
- `Istio sidecar` may also be the place where workload identity is sourced for a later `Service -> Online Keycloak` token request.
- `Online Keycloak` is used for online login and online token exchange.
- In the proposal variants, the intranet IAM is the exchange point for intranet tokens.
- The online service presents an online token to the intranet IAM, typically with JWT bearer authorization grant semantics and optionally with workload-based client authentication.
- For Keycloak-compatible workload identity, the `Service -> Online Keycloak` token request uses client authentication. In Keycloak 26.6.0 this can be a registered client secret, signed JWT, or federated client authentication backed by Kubernetes service account or another trusted issuer.

## Keycloak Support Notes

- `OIDC`: supported by current Keycloak.
- `DPoP`: supported in Keycloak 26.6.0.
- `JWT Authorization Grant`: supported in Keycloak 26.6.0 for external-to-internal token exchange using signed JWT assertions.
- `Standard token exchange (V2)`: supported in Keycloak 26.6.0, but only for internal-token to internal-token exchange in the same realm.
- `Federated client authentication`: supported in Keycloak 26.6.0 for OpenID Connect and Kubernetes service account assertions. SPIFFE support exists, but remains preview.
- `Delegation semantics`: current Keycloak does not document out-of-the-box support for RFC 8693 delegation features such as `actor_token` to `act` claim mapping.
- `DPoP subject tokens`: Keycloak standard token exchange rejects sender-constrained `subject_token` values, including DPoP-bound access tokens.
- `Workload identity at Keycloak`: for the online in-cluster hops in these proposals, workload identity is modeled as client authentication on the `Service -> Online Keycloak` token request. `JWT Authorization Grant` is a different mechanism and is only relevant when the external assertion itself is the authorization grant.
- `Intranet token exchange`: outside Keycloak standard token exchange V2. These proposals assume the intranet IAM accepts trusted online tokens using JWT bearer authorization grant semantics and optionally SPIFFE-based or JWT-assertion-based workload authentication.
