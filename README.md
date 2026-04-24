# Argo CD on EKS — deployment flow (diagrams)

Text runbooks: [eks-deploy.md](eks-deploy.md), [deploy-eks-keycloak.md](deploy-eks-keycloak.md). The diagrams below match this repo’s Terraform and typical EKS work.

> **Viewing the diagrams:** They use [Mermaid](https://mermaid.js.org/). GitHub, GitLab, many IDEs, and Notion render Mermaid. If you only see code blocks, paste into [Mermaid Live Editor](https://mermaid.live) or add a Mermaid extension to your viewer.

---

## 1) End-to-end: from prerequisites to a usable Argo CD

```mermaid
flowchart TB
  subgraph prep["1. Prereqs"]
    A[("EKS cluster\nexists")]
    B["Ingress controller\n(ALB or NGINX)"]
    C["aws eks update-kubeconfig\n+ kubectl works"]
    D["Argo URL decided\n+ DNS + TLS plan"]
  end

  subgraph tf["2. Terraform"]
    T1["Copy terraform.tfvars.example\nset bucket, region, argocd_domain, ingress..."]
    T2["terraform init -backend=false"]
    T3["terraform apply\n(S3 state bucket + Argo CD via Helm)"]
    T4["terraform init -backend-config=backend.hcl\n-migrate-state"]
  end

  subgraph net["3. Network and TLS"]
    N1["kubectl get ingress -n argocd\nget LB / hostname"]
    N2["DNS: argocd host →\nload balancer address"]
    N3["TLS: cert for host\n+ secret or cert-manager"]
  end

  subgraph use["4. Use Argo"]
    U1["Open Argo CD UI in browser\n(HTTPS, your argocd_domain)"]
    U2["Get initial admin\npassword (Secret)"]
    U3[("Optional: Keycloak / OIDC\n+ RBAC in tfvars")]
    U4[("Create Application(s)\n→ sync workloads")]
  end

  A --> T1
  B --> T1
  C --> T1
  D --> T1
  T1 --> T2 --> T3 --> T4 --> N1
  N1 --> N2 --> N3 --> U1
  U1 --> U2
  U2 --> U3
  U3 --> U4
  N3 --> U1
```

---

## 2) Terraform and state (first-time S3 path used by this repository)

This is the “bootstrap” loop: the bucket is created in the **same** apply that installs Argo; state starts **local**, then you **point** the backend at the new bucket and **migrate**.

```mermaid
flowchart LR
  subgraph p["Preparation"]
    I["terraform init\n-backend=false"]
  end

  subgraph a["Create resources"]
    S["S3 bucket for state\n+ backend.hcl written"]
    H["Argo CD Helm\nrelease"]
  end

  subgraph m["Move state to S3"]
    M2["terraform init\n-backend-config=backend.hcl\n-migrate-state"]
  end

  subgraph o["Ongoing"]
    M3["terraform plan / apply\nstate lives in S3"]
  end

  p --> a
  S --> H
  a --> m
  M2 --> o
```

---

## 3) Optional: UI login with Keycloak (after Argo is up)

```mermaid
flowchart TB
  K0["Keycloak: realm, client,\nredirect URI: .../auth/callback"]
  K1["Group mapper\ngroups claim"]
  T0["Set oidc_* in\ntfvars or Helm values"]
  T1["terraform apply"]
  A1["Test SSO login\nin Argo CD UI"]
  A2[("Optional: disable\nlocal admin user")]

  K0 --> K1 --> T0 --> T1 --> A1 --> A2
```

---

## 4) Linear quick checklist (same story, one column)

If you need a **simple sequence** list without the boxes:

1. EKS + ingress + kubeconfig.  
2. `terraform init -backend=false` → `apply` → `init -backend-config=backend.hcl -migrate-state`.  
3. DNS to ingress LB, TLS in place, open Argo over HTTPS.  
4. (Optional) Keycloak client + `terraform` OIDC + RBAC.  
5. **Applications** in Argo to deploy your apps to the cluster (or more clusters).  

For command-level detail, use [eks-deploy.md](eks-deploy.md) first, then [deploy-eks-keycloak.md](deploy-eks-keycloak.md) for SSO.
