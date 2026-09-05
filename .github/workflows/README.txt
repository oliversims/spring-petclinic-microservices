.github/workflows/ — what each file does
=======================================

This folder holds GitHub Actions for the spring-petclinic-microservices
application repo (the Java services). Platform deploy (Argo CD / Helm) lives
in petclinic-platform, not here.


FILES
-----

1) build-push.yml
   Push to main → build changed services (arm64) → Trivy (report only) →
   push to petclinic-dev/* ECR → dispatch petclinic-platform for image.tag.
   Secrets: AWS_ROLE_ARN, AWS_REGION, AWS_ACCOUNT_ID, PLATFORM_REPO_TOKEN.
   No kubectl/helm (Argo CD deploys).


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
