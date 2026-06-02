# Graph Report - .  (2026-06-02)

## Corpus Check
- 138 files · ~84,599 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 165 nodes · 202 edges · 24 communities (13 shown, 11 thin omitted)
- Extraction: 85% EXTRACTED · 14% INFERRED · 0% AMBIGUOUS · INFERRED: 29 edges (avg confidence: 0.83)
- Token cost: 177,199 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Flux Reconcile Ordering|Flux Reconcile Ordering]]
- [[_COMMUNITY_Keycloak Identity Stack|Keycloak Identity Stack]]
- [[_COMMUNITY_Backup Enablement & PITR|Backup Enablement & PITR]]
- [[_COMMUNITY_ASP Scraper Platform|ASP Scraper Platform]]
- [[_COMMUNITY_GLPI ITSM Stack|GLPI ITSM Stack]]
- [[_COMMUNITY_Monitoring Stack|Monitoring Stack]]
- [[_COMMUNITY_Renovate & Prod Overlays|Renovate & Prod Overlays]]
- [[_COMMUNITY_Cloudflare Tunnel & Keycloak Edge|Cloudflare Tunnel & Keycloak Edge]]
- [[_COMMUNITY_Audiobookshelf App|Audiobookshelf App]]
- [[_COMMUNITY_Linkding Bookmarks|Linkding Bookmarks]]
- [[_COMMUNITY_ASP-DB Backup (Production)|ASP-DB Backup (Production)]]
- [[_COMMUNITY_ASP-DB Backup (Staging)|ASP-DB Backup (Staging)]]
- [[_COMMUNITY_GLPI App Deployment|GLPI App Deployment]]
- [[_COMMUNITY_Cloudflare Staging|Cloudflare Staging]]
- [[_COMMUNITY_Bootstrap & Architecture Docs|Bootstrap & Architecture Docs]]
- [[_COMMUNITY_Flux Staging Root|Flux Staging Root]]
- [[_COMMUNITY_Project README|Project README]]
- [[_COMMUNITY_ASP Data PVC|ASP Data PVC]]
- [[_COMMUNITY_Audiobookshelf Prod Overlay|Audiobookshelf Prod Overlay]]
- [[_COMMUNITY_Monitoring Controllers (Prod)|Monitoring Controllers (Prod)]]
- [[_COMMUNITY_Monitoring Controllers (Staging)|Monitoring Controllers (Staging)]]
- [[_COMMUNITY_CNPG Staging Overlay|CNPG Staging Overlay]]
- [[_COMMUNITY_Staging Kubeconfig|Staging Kubeconfig]]
- [[_COMMUNITY_Devcontainer Setup|Devcontainer Setup]]

## God Nodes (most connected - your core abstractions)
1. `sops-age decryption SecretRef` - 8 edges
2. `Database Backups CNPG to Cloudflare R2` - 8 edges
3. `plugin-barman-cloud HelmRelease` - 7 edges
4. `ObjectStore r2-store` - 7 edges
5. `Keycloak Production Overlay` - 7 edges
6. `asp-db CNPG Cluster` - 6 edges
7. `orchestrator Deployment` - 6 edges
8. `audiobookshelf Deployment` - 6 edges
9. `Flux Kustomization apps path ./apps/staging` - 6 edges
10. `Keycloak StatefulSet` - 6 edges

## Surprising Connections (you probably didn't know these)
- `CNPG ObjectStore r2-store (Cloudflare R2)` --depends_on--> `Flux Kustomization infra-cnpg-plugin (production)`  [INFERRED]
  apps/staging/asp/objectstore.yaml → clusters/production/infrastructure.yaml
- `Flux Kustomization apps path ./apps/staging` --references--> `Kustomize overlay asp (staging)`  [INFERRED]
  clusters/staging/apps.yaml → apps/staging/asp/kustomization.yaml
- `Flux Kustomization apps path ./apps/staging` --references--> `Kustomize overlay audiobookshelf (staging)`  [INFERRED]
  clusters/staging/apps.yaml → apps/staging/audiobookshelf/kustomization.yaml
- `Flux Kustomization apps path ./apps/staging` --references--> `Kustomize overlay linkding (staging)`  [INFERRED]
  clusters/staging/apps.yaml → apps/staging/linkding/kustomization.yaml
- `Flux Kustomization apps path ./apps/production` --references--> `Kustomize overlay linkding (production)`  [INFERRED]
  clusters/production/apps.yaml → apps/production/linkding/kustomization.yaml

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **ASP scraping pipeline (crawler/scraper/orchestrator share DB)** — scraper_crawlerdeployment_urlcrawler, scraper_contentdeployment_contentscraper, orchestrator_deployment_orchestrator, asp_database_aspdb [INFERRED 0.85]
- **audiobookshelf persistent volumes** — audiobookshelf_deployment_audiobookshelf, audiobookshelf_storage_configpvc, audiobookshelf_storage_metadatapvc, audiobookshelf_storage_bookspvc [EXTRACTED 1.00]
- **asp CNPG R2 Backup Flow** — asp_cluster_asp_db, asp_objectstore_r2_store, asp_scheduledbackup_asp_db_daily, asp_objectstore_secret_r2_credentials [INFERRED 0.85]
- **glpi MariaDB Database Stack** — glpi_database_statfulset_db, glpi_database_service_db, glpi_database_configmap_db, glpi_database_secret_db, glpi_database_storage_db_pvc [INFERRED 0.85]
- **linkding App Stack** — linkding_deployment_linkding, linkding_service_linkding, linkding_storage_data_pvc, linkding_ingress_production [INFERRED 0.85]
- **Production Flux reconcile dependency chain** — production_infrastructure_infra_certmanager, production_infrastructure_infra_cnpg_plugin, production_infrastructure_infrastructure_controllers, production_apps_apps [EXTRACTED 1.00]
- **Staging Flux reconcile dependency chain** — staging_infrastructure_infra_certmanager, staging_infrastructure_infra_cnpg_plugin, staging_infrastructure_infrastructure_controllers, staging_apps_apps [EXTRACTED 1.00]
- **asp CNPG R2 backup overlay (Cluster, ObjectStore, ScheduledBackup, patch)** — staging_asp_cluster_asp_db, staging_asp_objectstore, staging_asp_scheduledbackup, staging_asp_cluster_backup_patch, staging_asp_r2_backup_credentials [EXTRACTED 1.00]
- **Backup enablement stack (cert-manager + plugin + operator)** — cert_manager_release_cert_manager, plugin_release_plugin_barman_cloud, cnpg_release_cnpg [INFERRED 0.85]
- **Keycloak app deployment unit** — keycloak_app_statefulset_keycloak, keycloak_app_configmap_keycloak_config, keycloak_app_service_keycloak, keycloak_app_service_keycloak_discovery [EXTRACTED 1.00]
- **Flux CRD gating order (certmanager to plugin to consumers)** — cert_manager_release_cert_manager, plugin_release_plugin_barman_cloud, backups_flux_order_concept [INFERRED 0.85]
- **Keycloak Production R2 Backup Flow** — database_cluster_keycloak_db, database_cluster_backup_patch_keycloak_db, database_objectstore_r2_store, database_scheduledbackup_keycloak_db_daily, database_secret_r2_backup_credentials [INFERRED 0.85]
- **Controllers Production Overlay Set** — production_kustomization_controllers, renovate_kustomization_production, keycloak_kustomization_production, cloudflare_kustomization_production, cnpg_kustomization_production [INFERRED 0.75]
- **Renovate CronJob Config Set** — renovate_cronjob_renovate, renovate_configmap_renovate, renovate_renovate_container_env_secret [INFERRED 0.85]
- **Staging Keycloak Database R2 Backup Flow** — database_cluster_keycloak_db, database_objectstore_r2_store, database_scheduledbackup_keycloak_db_daily, database_secret_r2_backup_credentials, database_cluster_backup_patch_keycloak [INFERRED 0.85]
- **kube-prometheus-stack Monitoring Stack** — kube_prometheus_stack_kustomization_base, kube_prometheus_stack_namespace_monitoring, kube_prometheus_stack_repository_helmrepository, kube_prometheus_stack_release_helmrelease [INFERRED 0.85]
- **Monitoring Staging/Production Overlays over Base** — kube_prometheus_stack_kustomization_controllers_staging, kube_prometheus_stack_kustomization_controllers_production, kube_prometheus_stack_kustomization_base [INFERRED 0.85]

## Communities (24 total, 11 thin omitted)

### Community 0 - "Flux Reconcile Ordering"
Cohesion: 0.10
Nodes (23): Rationale: Flux dependsOn reconcile ordering (certmanager before cnpg-plugin before controllers/apps), Flux Kustomization apps path ./apps/production, Flux GitRepository flux-system (production), Flux Kustomization flux-system path ./clusters/production, Flux Kustomization infra-certmanager (production), Flux Kustomization infra-cnpg-plugin (production), Flux Kustomization infrastructure-controllers (production), Kustomize overlay linkding (production) (+15 more)

### Community 1 - "Keycloak Identity Stack"
Cohesion: 0.17
Nodes (18): Staging Keycloak App Kustomization, Keycloak App Production Overlay, Staging Keycloak App Secret, Controllers Base Kustomization, keycloak-db Barman Backup Patch, Cluster Backup Patch (barman-cloud plugin), CNPG Cluster keycloak-db, Keycloak Database Kustomization (+10 more)

### Community 2 - "Backup Enablement & PITR"
Cohesion: 0.18
Nodes (17): Flux deployment ordering (CRD gating), ObjectStore CR (r2-store) backup config, Point-in-time recovery (PITR) design, R2 boto3 checksum-compatibility fix, cert-manager Kustomization, cert-manager Namespace, cert-manager HelmRelease, jetstack HelmRepository (+9 more)

### Community 3 - "ASP Scraper Platform"
Cohesion: 0.24
Nodes (15): asp-db CNPG Cluster, asp-db-app Secret (CNPG generated), asp-db-init ConfigMap, asp Kustomization, asp Namespace, orchestrator-config ConfigMap, orchestrator Deployment, orchestrator-secrets Secret (+7 more)

### Community 4 - "GLPI ITSM Stack"
Cohesion: 0.21
Nodes (13): glpi App Production Kustomization, glpi App Service, glpi Config PVC, glpi DB ConfigMap, glpi Database Kustomization, glpi Database Production Kustomization, glpi DB Secret, glpi DB Service (+5 more)

### Community 5 - "Monitoring Stack"
Cohesion: 0.18
Nodes (12): kube-prometheus-stack Base Kustomization, Production kube-prometheus-stack Config Kustomization, Staging kube-prometheus-stack Config Kustomization, Production kube-prometheus-stack Overlay Kustomization, Staging kube-prometheus-stack Overlay Kustomization, Namespace monitoring, kube-prometheus-stack HelmRelease, kube-prometheus-stack HelmRepository (+4 more)

### Community 6 - "Renovate & Prod Overlays"
Cohesion: 0.22
Nodes (11): Cloudflared ConfigMap (production), Cloudflare Production Overlay, CNPG Production Overlay, Controllers Production Kustomization, Renovate ConfigMap, Renovate CronJob, Renovate Base Kustomization, Renovate Production Overlay (+3 more)

### Community 7 - "Cloudflare Tunnel & Keycloak Edge"
Cohesion: 0.31
Nodes (10): cloudflared Deployment (tunnel), cloudflare Kustomization, cloudflare Namespace, Keycloak Identity & Scaling design, keycloak-config ConfigMap, Keycloak app Kustomization, keycloak Service (ClusterIP), keycloak-discovery headless Service (+2 more)

### Community 8 - "Audiobookshelf App"
Cohesion: 0.36
Nodes (8): audiobookshelf-configmap ConfigMap, audiobookshelf Deployment, audiobookshelf Kustomization, audiobookshelf Namespace, audiobookshelf Service, audiobookshelf-books-pvc PVC, audiobookshelf-config-pvc PVC, audiobookshelf-metadata-pvc PVC

### Community 9 - "Linkding Bookmarks"
Cohesion: 0.38
Nodes (7): linkding Deployment, linkding Admin User Secret, linkding Production Ingress (linkding.eliorion.fr), linkding Base Kustomization, linkding Namespace, linkding Service, linkding Data PVC

### Community 10 - "ASP-DB Backup (Production)"
Cohesion: 0.53
Nodes (6): CNPG Cluster asp-db, asp Cluster Backup Patch (Barman plugin), asp Production Overlay Kustomization, ObjectStore r2-store (Cloudflare R2), r2-backup-credentials Secret, ScheduledBackup asp-db-daily

### Community 11 - "ASP-DB Backup (Staging)"
Cohesion: 0.47
Nodes (6): CNPG Cluster asp-db, asp-db Cluster backup patch (barman-cloud WAL archiver), Kustomize overlay asp (staging), CNPG ObjectStore r2-store (Cloudflare R2), Secret r2-backup-credentials, CNPG ScheduledBackup asp-db-daily

### Community 12 - "GLPI App Deployment"
Cohesion: 0.50
Nodes (5): glpi-configmap ConfigMap, glpi-db (MySQL host ref), glpi Deployment, glpi-secret Secret, glpi app Kustomization

## Ambiguous Edges - Review These
- `Kustomize overlay linkding (staging)` → `Ingress linkding linkding.eliorion.fr`  [AMBIGUOUS]
  apps/staging/linkding/kustomization.yaml · relation: references

## Knowledge Gaps
- **56 isolated node(s):** `K3S VM Homelab README`, `asp Namespace`, `asp-data PVC`, `orchestrator-secrets Secret`, `audiobookshelf Namespace` (+51 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **11 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `Kustomize overlay linkding (staging)` and `Ingress linkding linkding.eliorion.fr`?**
  _Edge tagged AMBIGUOUS (relation: references) - confidence is low._
- **Why does `Database Backups CNPG to Cloudflare R2` connect `Backup Enablement & PITR` to `Cloudflare Tunnel & Keycloak Edge`?**
  _High betweenness centrality (0.016) - this node is a cross-community bridge._
- **Why does `Keycloak Production Overlay` connect `Keycloak Identity Stack` to `Renovate & Prod Overlays`?**
  _High betweenness centrality (0.015) - this node is a cross-community bridge._
- **Why does `Flux Kustomization apps path ./apps/staging` connect `Flux Reconcile Ordering` to `ASP-DB Backup (Staging)`?**
  _High betweenness centrality (0.015) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `sops-age decryption SecretRef` (e.g. with `SOPS age decryption config (production)` and `SOPS age decryption config (staging)`) actually correct?**
  _`sops-age decryption SecretRef` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `K3S VM Homelab README`, `asp Namespace`, `asp-data PVC` to the rest of the system?**
  _57 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Flux Reconcile Ordering` be split into smaller, more focused modules?**
  _Cohesion score 0.10276679841897234 - nodes in this community are weakly interconnected._