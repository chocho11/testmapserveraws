# MapServer on AWS — Full Infrastructure as Code (CloudFormation)

Ce projet déploie une architecture complète et scalable pour héberger **MapServer** sur AWS, entièrement automatisée via **CloudFormation**.  
L’objectif est de fournir une plateforme cartographique robuste, reproductible et auditable, avec séparation claire des composants réseau, sécurité, stockage et compute.

---

## 🚀 Architecture générale

L’infrastructure repose sur 4 stacks CloudFormation orchestrées par un **root stack** :
```bash
root.yml 
    ├── vpc.yml 
    ├── sg.yml 
    ├── efs.yml 
    └── lb-asg-tg.yml
```

Chaque stack est indépendante, exporte ses ressources critiques et peut être réutilisée dans d’autres environnements (dev, test, prod).

---

## 🧩 Composants principaux

### 1. **VPC (vpc.yml)**
- VPC 10.0.0.0/16  
- 2 subnets publics (AZ A/B)  
- 2 subnets privés (AZ A/B)  
- Internet Gateway  
- NAT Gateway pour les instances privées  
- Tables de routage dédiées  
- Exports : VPC ID, Subnets publics/privés

---

### 2. **Security Groups (sg.yml)**

| Security Group | Rôle | Règles |
|----------------|------|--------|
| **AlbAppSecurityGroup** | SG du Load Balancer | HTTP/HTTPS depuis Internet |
| **AlbMapserverSecurityGroup** | SG des instances MapServer | HTTP/HTTPS uniquement depuis le SG du LB |
| **EFSSecurityGroup** | SG du EFS | NFS (2049) uniquement depuis les instances MapServer |

Tous les SG sont exportés pour être réutilisés dans les autres stacks.

---

### 3. **EFS (efs.yml)**
- FileSystem chiffré  
- Mount Targets dans les 2 subnets privés  
- Sécurisé par le SG EFS  
- Export : FileSystemId  

L’EFS sert de **stockage partagé** pour :
- les données cartographiques  
- les fichiers `.map`  
- les shapefiles  
- les ressources statiques  

---

### 4. **Load Balancer + ASG + Launch Templates (lb-asg-tg.yml)**

#### 🔹 Application Load Balancer
- Listener HTTP (80)
- Listener HTTPS (443) optionnel
- Règles :
  - `/app.html` → Target Group Public
  - `/cgi-bin/mapserv*` → Target Group Private

#### 🔹 Auto Scaling Groups
- **ASG Public** : instances Apache servant `app.html`
- **ASG Private** : instances MapServer + montage EFS

#### 🔹 Launch Templates
- Installation automatique (Apache, MapServer, GDAL…)
- Montage EFS via `/etc/fstab`
- Initialisation automatique des données (shapefiles + mapfile)
- Activation du CGI MapServer
- Signalisation CloudFormation via `cfn-signal`

---

## 🌐 Fonctionnement global

1. L’utilisateur accède à :  
   **`http://<ALB-DNS>/app.html`**

2. La page Leaflet charge une couche WMS via :  
   **`http://<ALB-DNS>/cgi-bin/mapserv?map=/srv/nfs/mapfile/...`**

3. Le Load Balancer route automatiquement :
   - `/app.html` → instances publiques
   - `/cgi-bin/mapserv*` → instances privées MapServer

4. Les instances MapServer montent l’EFS et lisent :
   - les shapefiles
   - le fichier `.map`
   - les données partagées

---

## 🛠 Déploiement

### 1. Uploader les templates dans un bucket S3
```bash
- vpc.yml 
- sg.yml 
- efs.yml 
- lb-asg-tg.yml 
- root.yml
```
### 2. Déployer le root stack
```bash
# déploiement à partir du cloudshell de aws
         cd ~
         curl -L https://raw.githubusercontent.com/chocho11/testmapserveraws/main/stack/root.yml -o root.yml
         curl -L https://raw.githubusercontent.com/chocho11/testmapserveraws/main/stack/vpc.yml -o vpc.yml
         curl -L https://raw.githubusercontent.com/chocho11/testmapserveraws/main/stack/sg.yml -o sg.yml
         curl -L https://raw.githubusercontent.com/chocho11/testmapserveraws/main/stack/efs.yml -o efs.yml
         curl -L https://raw.githubusercontent.com/chocho11/testmapserveraws/main/stack/lb-asg-tg.yml -o lb-asg-tg.yml

         BUCKET="testmapserver23666666" # name of your S3 bucket

         aws s3 cp stack/ s3://$BUCKET/ --recursive # Uploading yml files to S3 bucket $BUCKET"

         aws cloudformation deploy --stack-name mapserver-root --template-file root.yml --parameter-overrides TemplateBucket=$BUCKET --capabilities CAPABILITY_NAMED_IAM
 ```  
 ```bash 
  📁 Structure du dépôt
/
├── vpc.yml
├── sg.yml
├── efs.yml
├── lb-asg-tg.yml
└── root.yml
``` 



🔒 Sécurité
- Instances MapServer non exposées à Internet
- Accès MapServer uniquement via l’ALB
- EFS accessible uniquement depuis les instances MapServer
- Séparation stricte public / privé
- IAM minimal (pas de rôles excessifs)

📌 Objectifs du projet
- Déployer MapServer de manière reproductible
- Fournir une architecture scalable et sécurisée
- Séparer proprement les responsabilités (réseau, compute, stockage)
- Faciliter l’audit et la maintenance
- Offrir une base solide pour des projets SIG plus avancés

📜 Licence
Libre d’utilisation et d’adaptation selon vos besoins.
 
🏛️ Architecture

```mermaid
flowchart TB

%% ============================
%% Styles AWS
%% ============================
classDef aws fill:#fefefe,stroke:#d5d5d5,stroke-width:1px,color:#333,border-radius:6px;
classDef group fill:#f2f2f2,stroke:#999,stroke-width:1px,color:#333,border-radius:8px;

%% ============================
%% VPC
%% ============================
subgraph VPC["VPC 10.0.0.0/16"]
class VPC group;

    subgraph PublicSubnets["Subnets Publics"]
    class PublicSubnets group;
        PSa["PublicSubnetA<br/>AZ A"]
        PSb["PublicSubnetB<br/>AZ B"]
    end

    subgraph PrivateSubnets["Subnets Privés"]
    class PrivateSubnets group;
        PRa["PrivateSubnetA<br/>AZ A"]
        PRb["PrivateSubnetB<br/>AZ B"]
    end
end

%% ============================
%% Security Groups
%% ============================
SG_ALB["SG ALB<br/>HTTP/HTTPS depuis Internet"]
SG_MS["SG MapServer<br/>HTTP/HTTPS depuis ALB"]
SG_EFS["SG EFS<br/>NFS 2049 depuis MapServer"]

class SG_ALB,SG_MS,SG_EFS aws;

%% ============================
%% Load Balancer
%% ============================
ALB["Application Load Balancer"]
class ALB aws;

TG_Public["Target Group Public<br/>/app.html"]
TG_Private["Target Group Private<br/>/cgi-bin/mapserv*"]
class TG_Public,TG_Private aws;

%% ============================
%% ASG Public (App)
%% ============================
ASG_Public["ASG Public<br/>Apache + app.html"]
LT_App["Launch Template App"]
class ASG_Public,LT_App aws;

%% ============================
%% ASG Private (MapServer)
%% ============================
ASG_Private["ASG Private<br/>MapServer + EFS"]
LT_MS["Launch Template MapServer"]
class ASG_Private,LT_MS aws;

%% ============================
%% EFS
%% ============================
EFS["EFS<br/>Stockage partagé"]
MTa["Mount Target A"]
MTb["Mount Target B"]
class EFS,MTa,MTb aws;

%% ============================
%% Internet
%% ============================
Internet(("Internet"))

%% ============================
%% Relations
%% ============================

Internet --> ALB
SG_ALB --> ALB

ALB -->|HTTP/HTTPS| TG_Public
ALB -->|/cgi-bin/mapserv*| TG_Private

TG_Public --> ASG_Public
ASG_Public --> LT_App
LT_App --> SG_ALB

TG_Private --> ASG_Private
ASG_Private --> LT_MS
LT_MS --> SG_MS

SG_MS --> SG_EFS
SG_EFS --> EFS

EFS --> MTa --> PRa
EFS --> MTb --> PRb
```