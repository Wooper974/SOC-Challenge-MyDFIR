# SOC-Challenge-MyDFIR
## Introduction
Bonjour à tous, dans le but de développer mes compétences techniques sur les différentes tâches que peut rencontrer un analyste SOC, j'ai participé au challenge organisé par le vidéaste MyDFIR.

Ce challenge s'est déroulé sur 30 jours et j'ai décidé de rédiger un compte-rendu qui va principalement aborder les points suivants :
   * Mise en place de l'infrastructure du lab
   * Création des alertes sur ELK
   * Génération d'un Payload Malveillant depuis notre C2 Server
   * Automatisation vers notre serveur de Ticketing

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

Nous souhaitons être capable de voir les logs Sysmon depuis notre console Elasticsearch, et il va falloir en premier lieu ajouter l'intégration suivante :  
![wineventlog](https://github.com/user-attachments/assets/fa39aae1-a113-4b57-a80a-9010a1f86cf3)

L'un des champs les plus importants lorsqu'il s'agit de configurer une intégration, est le champ "Channel Name", car c'est dans ce champ qu'on va spécifier le chemin qui contient les logs Sysmon dans l'observateur d'événements :
![channelname-sysmon](https://github.com/user-attachments/assets/14fffd3d-6da2-45fa-9833-68f6570fb32b)

On va littéralement faire la même manipulation pour les logs de Microsoft Defender sauf que cette fois-ci, on va devoir filtrer les events qu'on veut remonter car ils ne sont pas tous pertinents.  
Il faut deja commencer par ajouter le bon chemin dans le champ "Channel Name" qui est le suivant : Microsoft-Windows-Windows Defender/Operational.  
Et maintenant, on va s'intérésser au champ "Event ID" car c'est ici qu'on va pouvoir spécifier les events qu'on souhaite faire remonter :  
![eventid-defender](https://github.com/user-attachments/assets/baf686e4-bb25-4b91-89f7-4a4b4071ff80)

   * Event ID 1116 : Defender détécte un malware ou un logiciel potentiellement compromettant
   * Event ID 1117 : Defender a effectué une action pour protéger notre poste d'un malware ou d'un logiciel jugé suspicieux
   * Event ID 5001 : La fonctionnalitié de Defender permettant de scanner des malware sur notre poste a été désactivée.

Pour vérifier que les logs Sysmon remontent dans notre ELK, on peut rechercher par exemple l'event ID 1 qui correspond à la création de processus : 
![sysmonlog-discover](https://github.com/user-attachments/assets/b0b0be85-0273-424d-90a7-7dd87959f6a2)

## Installation et Monitoring du serveur Ubuntu

Ce serveur a la même objectif que notre serveur Windows, c'est à dire avoir un service exposé sur internet qui sera en l'occurence SSH.  
Après avoir installé la version 22.04 de Ubuntu, il faut simplement s'assurer que le service SSH soit bien installé et activé.

Il reste à installer l'agent Elastic sur ce serveur et comme précédemment, on va créer une nouvelle politique d'agent Linux.  
On souhaite collecter les logs permettant d'observer une attaque par bruteforce en passant par le service SSH, on aura donc uniquement besoin de collecter les informations du fichier /var/log/auth/log.

Ensuite, on va retrouver le même procéder que pour le serveur windows en créant un nouvelle agent, ce qui aura pour effet de nous générer une commande à entrer sur le serveur Linux pour effectuer l'installation de l'agent. ATTENTION : Il ne faut pas oublier de créer une règle de pare-feu qui autorise la communication entre notre nouveau serveur Linux et le serveur ELK
Pour vérifier que les logs de notre Machine Ubuntu remontent, vous devriez voir apparaître dans le champ "agent.name" le nom du serveur.

## Création des alertes sur ELK

On va créer une alerte qui est en soit relativement simple mais qui répond néanmoins à ce que recherchions, à savoir être alerter lorsqu'il y a potentiellement une attaque par brute force faite sur notre machine Windows et Linux qui ont respectivement le service RDP et SSH exposé sur internet.

Pour notre machine Linux, on va créer cette alerte qui sera déclenché dès qu'il y aura plus de 3 tentatives de connexions par SSH à notre machine en l'espace de 2 minutes :  
![alert-sshbruteforce](https://github.com/user-attachments/assets/91c857eb-ce41-45bb-bfbc-40db863ef85f)

Pour notre machine Windows, on utilise la même approche adaptée pour le service RDP car on va regarder cette fois-ci l'event ID 4625 qui signifie qu'il y a eu un compte qui n'a pas réussi à s'authentifier, et donc potentiellement une attaque par bruteforce : 
![alert-rdp brute force](https://github.com/user-attachments/assets/5c21ec62-cec0-4d5d-93d2-4ea5df4e1a2a)

Les deux règles que nous venons de créer ne sont toutefois pas aussi optimales que nous le souhaitons car en l'état, si vous regardez comment sont affichés les alertes qui remontent, l'affichage des informations pertinentes sont loin d'être facilement visible.  
Nous allons donc maintenant créer des règles de détéctions en séléctionnant les champs qui vont fortement nous intérésser, à savoir :  
   * L'adresse source qui initie la tentative de connexion en SSH ou RDP
   * Le nom d'utilisateur qui tente de s'authentifier

Dans Security>Rules>Detection rules, on a la possibilité de créer des règles de détéction un peu plus customizable en spécifiant les champs qui nous intéressent : 
![Capture d’écran 2024-10-07 181552](https://github.com/user-attachments/assets/72eb7fa1-a977-4a17-b98d-2a1fdcde7f22)

## Mise en place de l'environnement de l'attaquant

Tout d'abord, l'attaquant utilisera comme technique ce qu'on appelle une attaque par Command & Control, qui peut se traduite par le fait qu'un attaquant tente de contrôler le système de la victime via plusieurs moyens. 
Ici, nous utiliserons un C2 server de manière à pouvoir injecter un fichier malveillant sur la machine de la victime, ce qui nous permettra par la suite d'avoir plus de privélége sur celle-ci afin de pouvoir réaliser des actions comme exfiltrer un fichier de la victime contenant des données confidentiels vers notre serveur.

### Installation et configuration du C2 Server

On va premièrement installer une distribution Linux basique type Ubuntu 22.04 pour ensuite installer Docker sur cette machine.

J'ai décidé d'utiliser le framework Mythic comme C2 server, et on va donc le téléchargé directement depuis son repositery avec la commande git clone https://github.com/its-a-feature/Mythic.  
Une fois dans le dossier Mythic, on va lancer le script qui permet de l'installer sur Docker via 2 séries de commande 
   * ./install_docker_ubuntu.sh
   * make
On maintenant lancer mythic avec la commande suivante : ./mythic_cli start  

Après avoir démarré Mythic, on va tout de suite se diriger vers son interface web (https:AdresseIP:7443).  
INFO : Les identifiants de connexion se trouve dans le fichier .env  

Nous avons désormais un C2 server fonctionnel qu'on va pouvoir utiliser pour parvenir à notre objectif de compromission.

### Phase 1 : Initial Access

Cette première phase a pour objectif d'obtenir un premier accès sur la machine victime depuis le poste attaquant.
Ce lab n'a pas pour but de se concentrer sur cette axe offensive donc nous allons établir un scénario assez simpliste où l'utilisateur de la machine windows a définit un mot de passe faible pour son compte Administrateur (Snk123!).  
Ensuite, on souhaite avoir un accès en RDP à la machine victime, on va donc faire faire une attaque par bruteforce depuis notre machine kali en ajoutant le mot de passe précédemment cité à notre wordlist :  
![rddddp successsssss](https://github.com/user-attachments/assets/992a926d-ac12-4479-baaf-8c5525f903a2)

### Phase 2 : Defense Evasion

On va ensuite désactiver les défenses de la machine windows pour faciliter l'exfiltration vers notre C2 server.
Ici, nous allons principalement désactiver Windows Defender et plus particulièrement sa protection en temps réel.


### Phase 3 : Initialisation de l'agent Mythic (Execution)

Nous allons maintenant générer un agent Mythic qui va permettre à la machine windows d'établir une connexion avec notre C2 server.

Tout d'abord, il faut télécharger un agent qui soit compatible avec un OS Windows. La page Github de Mythic mets à disposition une liste des différents agents disponibles et nous allons choisir d'installer l'agent Apollo via cette commande : 
![apollo dl](https://github.com/user-attachments/assets/e9d2bd44-13f0-4c92-9b1b-09d8292785a6)

Ensuite, il faut également télécharger un "C2 Profile" et celui que nous allons choisir dans notre cas est le profil http via cette commande : ./mythic-cli install https://github.com/MythicC2Profiles/http

On peut maintenant observer les deux composants qui vont nous servir pour générer notre payload : 
![apollo online](https://github.com/user-attachments/assets/1cd2a290-c46d-4beb-8e47-ebbf0214fc12) ![c2 profiles mythric](https://github.com/user-attachments/assets/29f9de93-cfcb-402b-99d7-72fc09d1bde4)

Dans Payload>Generate New Payload, on va utiliser les paramètres suivants :  
   * OS : Windows
   * Type : WinExe
   * Commands : Inclure toutes les commandes
   * C2 Profiles : http  
ATTENTION : il faut modifier le champ "Callback host" car nous souhaitons passer par http, il faut aussi renseigner l'adresse IP Publique de notre C2 Server :  
![c2 profiles](https://github.com/user-attachments/assets/2071c49a-4ec8-4aa9-b062-a33a2c4d2a7b)

Et enfin, on peut donner un nom à notre payload :  
![svchost-wooper](https://github.com/user-attachments/assets/33d2bd88-a083-410d-9db2-fd3c7a2d5638)

Nous pouvons maintenant copier le lien du payload pour l'importer sur notre C2 server, puis on va le renommer histoire de le rendre un peu plus digeste à lire :  
![rename payload on mythic](https://github.com/user-attachments/assets/f76668b8-c92a-4553-a62d-60eb5ab5803d)  

Pour rappel, l'objectif de cette phase est d'éxécuter notre payload sur la machine victime, et c'est dans cette démarche qu'on va lancer un serveur http sur le port 9999 depuis notre C2 server avec la commande : python3 -m http.server 9999.  

On peut maintenant passer par notre session RDP ouverte depuis le poste de l'attaquant, pour télécharger ce payload via la commande :  
![download on windows](https://github.com/user-attachments/assets/dbfef8b7-94db-4a99-87e6-878076579d43)  

Une fois le payload téléchargé, on peut tout de suite le démarrer avec la commande ./svchost-wooper.exe  
Pour vérifier que le payload fonctionne, si on retourne sur l'interface web de Mythic on devrait apercevoir une session actif dans l'onglet "Active Callbacks".  

### Phase 4 : Exfiltration  

Maintenant que nous avons une session actif depuis notre interface Mythic, on peut éxécuter un certain nombre de commande, et la commande qui va nous intérésser est celle qui va permettre l'exfiltration du fichier passwords.txt :  
![apollo command download pasdswd](https://github.com/user-attachments/assets/2be0d7d6-3b39-408f-8d1b-486315c8184e)

## Détéction d'une activité Mythic sur ELK

Cette phase d'attaque étant désormais terminé, en tant qu'analyste SOC nous souhaitons avant tout savoir comment détécter ce genre d'activité et dans notre cas, détécter la présence de Mythic sur les machines de notre parc.

Imaginons que nous voulons créer une alerte pour détécter si le payload malveillant est présent sur d'autres machines du parc. On va d'abord lister les champs qui peuvent nous aider en regardant les logs générés par l'éxécution de notre payload :  

   * event.code : 1
   * winlog.event_data.Hashes
   * winlog.event_data.OriginalFileName  

Pour rappel, L'event ID 1 représente dans sysmon la création d'un processus.  
Ensuite, nous pouvons également copier la signature SHA256 du payload et même si c'est relativement simple pour un attaquant de modifier la signature de son payload, ici on souhaite uniquement détécter ce payload en particulier.  
Et enfin, le champ winlog.event_Data.OriginalFileName contient le nom de l'agent utilisé dans notre payload, à savoir Apollo.  
![Capture d’écran 2024-10-19 202334](https://github.com/user-attachments/assets/dcaae9e3-cca8-4ca9-a852-1ada81114fbe)








