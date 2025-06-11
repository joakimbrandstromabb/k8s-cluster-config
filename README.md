# k8s-cluster-config
GitOps source of truth for Kubernetes cluster


Folder
What lives there
Why it’s outside the others
bootstrap/
Only what’s needed to bootstrap Argo CD
Lets you install Argo CD with a single kubectl apply -k bootstrap command. After that, Argo CD takes over.
clusters/
Everything Argo CD manages after it’s up
Keeps multi-cluster support simple—just add another folder for the next cluster.
infrastructure/ vs apps/
“Day-1” vs “Day-2”
Infra (Calico, Longhorn, MetalLB, cert-manager…) is foundational and rarely changes; apps change more often.
env/
Thin overlays
Each overlay only patches values that differ (replicas, ingress hostnames, etc.). Kustomize will merge those with the base. 
charts/
Optional public Helm charts
You can publish this folder as its own repo later without touching the GitOps tree.


That gives you a solid homelab base—Cilium networking, MetalLB load-balancer,
Longhorn storage, and NGINX ingress—all fully GitOps-managed by Argo CD.
Next steps?  Add cert-manager + Keycloak, or start pushing your own apps under
Kubernetes/apps/services/. Just ask when you’re ready.