.github/workflows/ — what each file does
=======================================

This folder holds GitHub Actions for the spring-petclinic-microservices
application repo (the Java services). Platform deploy (Argo CD / Helm) lives
in petclinic-platform, not here.


FILES
-----

1) build-push.yml
   Main (and only image) CI.
   On push to main, detects which of the 8 services changed, builds each as
   linux/arm64 (Maven buildDocker + QEMU), scans with Trivy (reports only;
   does not block push), pushes to ECR under petclinic-dev/<service>:<sha>
   (Terraform-created repos only), then repository_dispatch to petclinic-platform
   so helm-values image.tag can be updated.
   AWS: OIDC (no access keys). Secrets: AWS_ROLE_ARN, AWS_REGION, AWS_ACCOUNT_ID,
   plus PLATFORM_REPO_TOKEN for dispatch to petclinic-platform.
   Does NOT run kubectl/helm — Argo CD deploys.

2) check-pr-template.yml
   PR hygiene for this sample/fork (not related to ECR).
   On PR open/edit: closes obvious practice/bootcamp PRs; otherwise requires
   a filled PR template (needs-information label).
   Daily cron: closes PRs still incomplete after 7 days.
   Optional secret: ORG_READ_PAT (bypass private org members).


WORKFLOW SKETCH
---------------

  Developer
      |
      |  push to main (app repo)
      v
  +--------------------------------------------------+
  | build-push.yml                                   |
  |  1. path-filter → which services changed         |
  |  2. matrix: mvn buildDocker (arm64)              |
  |  3. Trivy scan                                   |
  |  4. push → ECR petclinic-dev/* (Terraform)       |
  |  5. repository_dispatch → petclinic-platform     |
  +------------------------+-------------------------+
                           |
                           v
  +--------------------------------------------------+
  | petclinic-platform (separate repo)               |
  |  update-image-tags.yml → commit helm-values TAG  |
  |  Argo CD ApplicationSet syncs → EKS              |
  +--------------------------------------------------+


ORDER FOR FIRST LIVE DEPLOY (manual reminder)
---------------------------------------------
  Images must exist in ECR before pods become Ready.
  Either run build-push (CI) or build/push locally (arm64), then set the
  same tag in petclinic-platform helm-values and let Argo sync.
