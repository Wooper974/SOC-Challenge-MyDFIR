# SOC-Challenge-MyDFIR
## Introduction
Bonjour à tous, dans le but de développer mes compétences techniques sur les différentes tâches que peut rencontrer un analyste SOC, j'ai participé au challenge organisé par le vidéaste MyDFIR.

Ce challenge s'est déroulé sur 30 jours et a pour but principal de faire monter en compétence les aspirants analyste SOC, en proposant des cas concrets.

## Topologie du challenge
![finalfinaifnal drawio](https://github.com/user-attachments/assets/6f5e1890-8df4-4b88-8313-1ebf1bf1c9ad)


Nous utiliserons l'hébergeur cloud Vultr pour installer notre environnement, en commençant par créer notre réseau virtuel privé selon l'adressage que j'ai choisi sur la topologie : 

![VPC](https://github.com/user-attachments/assets/56972d4b-fb05-46c7-97fe-f5478007ed64)

## Installation du serveur ELK

Premièrement, nous devons déployer la machine virtuelle qui fera office de serveur ELK.
J'ai pour ma part décidé de partir sur un environnement Ubuntu en faisant bien attention à placer la machine dans le réseau virtuel privé que nous avons mis en place précédemment :

![ELK Server](https://github.com/user-attachments/assets/68a1a64b-443e-4c74-a73a-9197ef313bad)

Une fois la machine lancée, on va se connecter en SSH à celle-ci, mais avant d'effectuer cette étape on va faire en sorte que seul notre PC puisse accéder à la machine, en autorisant uniquement notre adresse IP publique.
La plupart des hébergeurs cloud proposent comme fonctionnalité d'intégré notre machine virtuelle à un groupe de pare-feu, qui aura comme règle la suivante : 

![fw rule](https://github.com/user-attachments/assets/8e8d6fa4-430b-419d-b3ec-9e1a12c6792f)

Une fois connecté en SSH, on peut désormais commencer à installer les différentes composantes de la stack ELK :

### Elasticsearch

Après avoir installé la dépendance Elasticsearch, nous devons modifier certaines lignes dans le fichier /etc/elasticsearch/elasticsearch.yml.

On doit décommenter la ligne network.host pour entrer l'adresse IP publique du serveur, car par défaut Elasticsearch est uniquement accessible depuis le localhost, puis nous devons également décommenter la ligne http.port : 

![yml](https://github.com/user-attachments/assets/27cb430b-abfd-47f0-9f3a-894857a14886)

### Kibana

Au même titre qu'Elasticsearch, on va modifier quelques lignes dans le fichier de configuration de Kibana se trouvant au chemin /etc/kibana/kibana.yml.

Il faut décommenter la ligne server.host pour y inscrire à nouveau notre adresse IP publique, puis décommenter également la ligne server.port :  

![kiabanyml](https://github.com/user-attachments/assets/cc6bfe97-3a36-40a7-9fb8-7e1056909b0c)

Avant de partir sur l'interface web de Kibana, nous allons créer un token elasticsearch pour permettre la communication avec Kibana : 

![token](https://github.com/user-attachments/assets/c480576e-04d7-4a2e-8b74-1a83f9d49b0f)

On va maintenant se connecter à l'interface web de Kibana sur le port 5601, mais pour que cette opération fonctionne, il va falloir au préalable autoriser l'accès depuis notre PC vers l'ip publique du serveur sur ce même port.
2 Modifications sont nécessaires pour autoriser ce flux :

   * Règle de pare-feu autorisant le flux entrant de notre PC sur le port 5601
   * Règle de pare-feu sur le serveur ELK autorisant les flux entrants sur le port 5601

  ![allow 5601](https://github.com/user-attachments/assets/0a409eaf-ac62-42f5-863d-50c983b40033)
![Capture d’écran 2024-09-04 180637](https://github.com/user-attachments/assets/d8c26cec-6f3b-4086-b220-fd879c4471d0)

L'accès à l'interface web est désormais disponible et la première chose qui nous ais demandées avant de pouvoir naviguer sur l'interface, consiste à rentrer le token généré précédemment pour permettre la connexion de notre Elasticsearch sur Kibana :

![kibana-token](https://github.com/user-attachments/assets/432eb734-a912-48eb-943b-f8bc3b5addba)

## Installation du serveur Windows

Pour les besoins de ce lab, nous allons volontairement créer un serveur Windows 2022 qui aura le service RDP exposé sur internet.
  Conformément à la topologie du challenge, cette machine ne sera pas placée dans le réseau virtuel privé, car dans le cas où cette machine serait compromise, l'attaquant aura la possibilité de communiquer avec l'entièreté des autres machines du réseau, ce qui n'est clairement pas souhaitable.

A la création du serveur windows, il faut simplement ne séléctionner aucun groupe de pare-feu, de manière à autoriser toutes les demandes de connexion en bureau à distance (RDP) en provenance d'internet.
Une fois la machine créée, on va s'assurer que le service RDP est bien exposé sur internet en entrant l'adresse IP publique du serveur windows depuis notre PC : 

![winservexpose](https://github.com/user-attachments/assets/89b0ecaa-e624-45dd-9829-40c5a03aadbb)


## Installation du serveur Fleet

Tout d'abord un Fleet est un serveur qui va centraliser la gestion des agents Elastic, et un peu de la même manière que les GPO sur un AD, il va permettre de plus facilement pousser les nouvelles configurations sur tous les agents de votre parc informatique.

J'ai dans un premier temps créer une nouvelle machine virtuelle Ubuntu dans le même réseau virtuel privé que le serveur ELK.  
Ensuite, nous devons faire en sorte que le serveur ELK puisse savoir que nous souhaitons désigner ce nouveau serveur en tant que Fleet, et dans l'onglet Management>Fleet, on a la possibilité d'ajouter un serveur fleet en renseignant dans notre cas, l'adresse IP Publique de la dernière machine virtuelle créée.

Il ne nous reste plus qu'à entrer sur le serveur Fleet la commande qui a été générée suite à la précédente manipulation :

![fleet-generate](https://github.com/user-attachments/assets/76210ec3-88df-40d4-b595-5bd7bfefe163)

ATTENTION : Il ne faut pas oublier d'autoriser les flux entrants et sortants entre le serveur Fleet et le serveur ELK.

## Monitoring de la machine Windows

Maintenant que le serveur Fleet est configuré, on peut passer à l'ajout de l'agent elastic sur notre machine windows, et pour cette étape il faut d'abord créer une nouvelle politique d'agent. Cette politique aura pour effet de générer la série de commande à utiliser pour installer l'agent sur un hôte Windows : 

![Win-policy](https://github.com/user-attachments/assets/a2d0f74a-7f7a-498e-9b6e-f0fd1e9ba709)

Si vous entrez directement cette série de commande, vous allez vite remarquer qu'elle n'aboutit pas avec un message d'erreur à la clé.  
Il va falloir faire un peu de troubleshooting pour parvenir à nos fins.  

Premièrement, la documentation elasticsearch précise que la communication entre un agent et le serveur fleet se passe sur le port 8220, on va donc autoriser sur le serveur Fleet la communication sur ce port avec la commande ufw allow 8220.  
Il va falloir ensuite renseigner sur l'interface web d'Elasticsearch qu'on souhaite que le serveur Fleet écoute sur ce port, dans Management>Fleet>Settings :

![Capture d'écran 2024-09-21 110442](https://github.com/user-attachments/assets/45fa7a41-f5ed-4a0c-bc4a-55efb56af823)

Enfin, il faut ajouter la commande "--insecure" à la fin de la dernière commande car cette option permet d'autoriser le fait qu'on passe par un certificat auto-signée.  
Nous pouvons maintenant voir que notre serveur Windows remonte bien dans l'interface d'Elasticsearch :

![success-windows-fleet](https://github.com/user-attachments/assets/824c148d-0cb5-4681-b040-a022556e9e0c)

Il reste une dernière chose à faire pour terminer cette étape qui à terme doit permettre la remonter de logs du serveur windows sur notre serveur ELK, car jusqu'à maintenant nous avons fait en sorte que notre serveur windows puisse communiquer avec le serveur Fleet, mais il reste encore quelques autorisation du flux à faire pour permettre la communication avec le serveur ELK.  

Il va falloir s'assurer de deux choses :

   * Autoriser sur le pare-feu interne du serveur ELK qu'il autorise les communications entrantes sur le port 9200
   * Autoriser sur le pare-feu de notre hébérgeur, la communication entre notre serveur windows et le serveur ELK

Pour vérifier le bon fonctionnement de ces étapes, vous devriez pouvoir observer des logs remontés par le serveur windows dans l'onglet Analytics>Discover.  

J'ai par la suite installé Sysmon sur mon serveur windows qui me permettra d'avoir d'avantages de logs pertinents qui seront remontés sur Elasticsearch.

## Injection des logs Sysmon et Defender sur Elasticsearch


