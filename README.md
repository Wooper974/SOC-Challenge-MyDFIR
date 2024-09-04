# SOC-Challenge-MyDFIR
## Introduction
Bonjour à tous, dans le but de développer mes compétences techniques sur les différentes tâches que peut rencontrer un analyste SOC, j'ai participé au challenge organisé par le vidéaste MyDFIR.

Ce challenge s'est déroulé sur 30 jours et a pour but principal de faire monter en compétence les aspirants analyste SOC, en proposant des cas concrets.

## Topologie du challenge
![SOC-Challenge2](https://github.com/user-attachments/assets/6be469de-75cb-4b1a-9614-f945b275b964)

Nous utiliserons l'hébergeur cloud Vultr pour installer notre environnement, en commençant par créer notre réseau virtuel privé selon l'adressage que j'ai choisi sur la topologie: 

![VPC](https://github.com/user-attachments/assets/56972d4b-fb05-46c7-97fe-f5478007ed64)

## Installation du serveur ELK

Premièrement, nous devons déployer la machine virtuelle qui fera office de serveur ELK.
J'ai pour ma part décidé de partir sur un environnement Ubuntu en faisant bien attention à placer la machine dans le réseau virtuel privé que nous avons mis en place précédemment :

![ELK Server](https://github.com/user-attachments/assets/68a1a64b-443e-4c74-a73a-9197ef313bad)

Une fois la machine lancée, on va se connecter en SSH à celle-ci, mais avant d'effectuer cette étape on va faire en sorte que seul notre PC puisse accèder à la machine, en autorisant uniquement notre adresse IP Publique.
La plupart des hébérgeurs cloud propose comme fonctionnalité d'intégré notre machine virtuelle à un groupe de Pare-Feu, qui aura comme règle la suivante : 

![fw rule](https://github.com/user-attachments/assets/8e8d6fa4-430b-419d-b3ec-9e1a12c6792f)

Une fois connecté en SSH, on peut désormais commencer à installer les différentes composantes de la stack ELK :

### Elasticsearch

Après avoir installé la dépendance Elasticsearch, nous devons modifier certaines lignes dans le fichier /etc/elasticsearch/elasticsearch.yml.

On doit décommentez la ligne network.host pour entrer l'adresse IP Publique du serveur car par défaut elasticsearch est uniquement accessible depuis le localhost, puis nous devons également décommentez la ligne http.port : 

![yml](https://github.com/user-attachments/assets/27cb430b-abfd-47f0-9f3a-894857a14886)

### Kibana

Au même titre qu'elasticsearch, on va modifier quelques lignes dans le fichier de configuration de Kibana se trouvant au chemin /etc/kibana/kibana.yml.

Il faut décommenter la ligne server.host pour y inscrire à nouveau notre adresse IP publique, puis décommentez également la ligne server.port :  

![kiabanyml](https://github.com/user-attachments/assets/cc6bfe97-3a36-40a7-9fb8-7e1056909b0c)

Avant de partir sur l'interface web de Kibana, nous allons créer un token elasticsearch pour permettre la communication avec Kibana : 

![token](https://github.com/user-attachments/assets/c480576e-04d7-4a2e-8b74-1a83f9d49b0f)

On va maintenant se connecter à l'interface web de Kibana sur le port 5601, mais pour que cette opération fonctionne, il va falloir au préalable autoriser l'accès depuis notre pc vers l'ip publique du serveur sur ce même port.
2 Modifictions sont nécéssaires pour autoriser ce flux :
    * Règle de pare-feu autorisant le flux entrant de notre PC sur le port 5601
    * Règle de pare-feu sur le serveur ELK autorisant les flux entrants sur le port 5601

![allow 5601](https://github.com/user-attachments/assets/d99e5f3b-8aa2-40da-ab09-6282dd61a807)    ![Capture d’écran 2024-09-04 180407](https://github.com/user-attachments/assets/4a60151d-a698-4ade-b44b-8309f759b322)

