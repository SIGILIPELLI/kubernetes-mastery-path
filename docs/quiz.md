---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Kubernetes Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "You deploy a bare Pod for a real workload and its node dies. What happens?",
    "options": [
      "It restarts on another node automatically",
      "Nothing recreates it; use a Deployment for self-healing",
      "The Service recreates it",
      "The ReplicaSet recreates it"
    ],
    "correctIndex": 1,
    "explanation": "A bare Pod has no controller behind it."
  },
  {
    "question": "You change a Deployment's image tag. What does the Deployment controller do?",
    "options": [
      "Creates a new ReplicaSet and gradually shifts replicas from the old one (rolling update)",
      "Does nothing until restarted",
      "Deletes all Pods at once",
      "Edits Pods in place"
    ],
    "correctIndex": 0,
    "explanation": "The old ReplicaSet is kept at zero for rollback."
  },
  {
    "question": "A rollout is broken. Which command returns to the previous revision?",
    "options": [
      "kubectl rollout undo deployment/web",
      "kubectl scale deployment web --replicas=0",
      "kubectl delete deployment web",
      "kubectl rollback web"
    ],
    "correctIndex": 0,
    "explanation": "rollout undo reverts to the previous revision; --to-revision picks a specific one."
  },
  {
    "question": "Pods get new IPs on each recreation, but other services must reach them reliably. What do you create?",
    "options": [
      "An Ingress only",
      "A ConfigMap",
      "A Service with a label selector",
      "A hard-coded IP list"
    ],
    "correctIndex": 2,
    "explanation": "A Service gives a stable IP and DNS name in front of matching Pods."
  },
  {
    "question": "You need one external domain with path-based routing to ten HTTP services and TLS. What do you use?",
    "options": [
      "An Ingress plus an Ingress controller",
      "Ten NodePorts",
      "A ConfigMap",
      "Ten LoadBalancer Services"
    ],
    "correctIndex": 0,
    "explanation": "Ingress describes host/path routing; the controller implements it."
  },
  {
    "question": "A database password must not be baked into the image or stored in plain config. What do you use?",
    "options": [
      "A ConfigMap",
      "A Secret consumed as an env var or mounted file",
      "A label",
      "An annotation"
    ],
    "correctIndex": 1,
    "explanation": "Secrets are for sensitive data (base64-encoded, optionally encrypted at rest)."
  },
  {
    "question": "A container is deadlocked but its process is still running. Which probe restarts it?",
    "options": [
      "Readiness probe",
      "Startup probe",
      "Liveness probe",
      "No probe can"
    ],
    "correctIndex": 2,
    "explanation": "A failing liveness probe makes kubelet kill and restart the container."
  },
  {
    "question": "A Pod is temporarily unable to serve because its database connection dropped. What is the right probe response?",
    "options": [
      "Failing readiness removes it from Service endpoints without a restart",
      "Delete the Pod",
      "Failing liveness restarts it",
      "Scale the Deployment to zero"
    ],
    "correctIndex": 0,
    "explanation": "Readiness protects traffic without needless restarts."
  },
  {
    "question": "A Pod shows OOMKilled. What happened?",
    "options": [
      "The node was drained",
      "It exceeded its memory limit and the kernel OOM killer terminated it",
      "It exceeded its CPU limit",
      "The image failed to pull"
    ],
    "correctIndex": 1,
    "explanation": "Memory limits are enforced by killing; CPU limits by throttling."
  },
  {
    "question": "A Pod is stuck Pending with 'Insufficient cpu'. What does it tell you?",
    "options": [
      "The image is wrong",
      "A PVC is unbound",
      "The probe failed",
      "No node has enough unrequested CPU for its requests"
    ],
    "correctIndex": 3,
    "explanation": "The scheduler compares requests to allocatable capacity."
  },
  {
    "question": "A pod shows CrashLoopBackOff. Which command shows why the previous container crashed?",
    "options": [
      "kubectl top pod",
      "kubectl get pod",
      "kubectl delete pod",
      "kubectl logs <pod> --previous"
    ],
    "correctIndex": 3,
    "explanation": "--previous shows the crashed container's logs."
  },
  {
    "question": "You want to run a database migration once, retry on failure and stop. What kind of object?",
    "options": [
      "A bare Pod with Always restart",
      "A Job (restartPolicy Never or OnFailure)",
      "A DaemonSet",
      "A Deployment"
    ],
    "correctIndex": 1,
    "explanation": "A Job runs to completion; backoffLimit caps retries."
  },
  {
    "question": "A database needs stable Pod names, stable DNS and its own PVC per replica. What do you use?",
    "options": [
      "A DaemonSet",
      "A StatefulSet with a headless Service",
      "A Deployment",
      "A Job"
    ],
    "correctIndex": 1,
    "explanation": "StatefulSets give each replica a stable identity."
  },
  {
    "question": "You want a service account to read Pod logs only in the prod namespace. What do you create?",
    "options": [
      "A NetworkPolicy",
      "A ConfigMap",
      "A Role and a RoleBinding in prod",
      "A ClusterRoleBinding to admin"
    ],
    "correctIndex": 2,
    "explanation": "Role + RoleBinding grants permissions within one namespace."
  },
  {
    "question": "By default any Pod can reach any other Pod. What is the recommended first NetworkPolicy for a namespace?",
    "options": [
      "Open all ports",
      "Delete the CNI",
      "Default-deny, then allow-list the traffic you need",
      "Allow all"
    ],
    "correctIndex": 2,
    "explanation": "Policies are additive; start with deny-all."
  },
  {
    "question": "You want replicas to scale with CPU, keeping average utilization at 70%. What do you use, and what must be set?",
    "options": [
      "VerticalPodAutoscaler only",
      "A Job",
      "A ConfigMap",
      "HorizontalPodAutoscaler, with accurate CPU requests and metrics-server"
    ],
    "correctIndex": 3,
    "explanation": "Utilization is a percentage of the Pod's CPU request."
  },
  {
    "question": "You must stop a node drain from evicting too many replicas of your app at once. What protects availability?",
    "options": [
      "A ResourceQuota",
      "A PodDisruptionBudget",
      "A taint",
      "A liveness probe"
    ],
    "correctIndex": 1,
    "explanation": "PDBs limit voluntary disruptions."
  },
  {
    "question": "You want to reconcile cluster state continuously from Git, reverting manual edits. What approach and settings?",
    "options": [
      "A cron job",
      "kubectl apply from a laptop",
      "Manual edits",
      "GitOps (ArgoCD) with prune and selfHeal"
    ],
    "correctIndex": 3,
    "explanation": "The pull model corrects drift automatically."
  },
  {
    "question": "You must reject Pods that run as root or lack resource limits across a namespace. Which mechanisms apply?",
    "options": [
      "A Service",
      "A ConfigMap",
      "Pod Security labels (restricted) and admission policy",
      "An HPA"
    ],
    "correctIndex": 2,
    "explanation": "Admission control checks the object's content."
  }
]
</script>
