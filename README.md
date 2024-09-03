# SOC-Challenge-MyDFIR
## Introduction
Bonjour à tous, dans le but de développer mes compétences techniques sur les différentes tâches que peut rencontrer un analyste SOC, j'ai participé au challenge organisé par le vidéaste MyDFIR.

Ce challenge s'est déroulé sur 30 jours et a pour but principal de faire monter en compétence les aspirants SOC analyste, en proposant des cas concrets.

## Topologie du challenge
![SOC-Challenge2](https://github.com/user-attachments/assets/6be469de-75cb-4b1a-9614-f945b275b964)

Nous utiliserons l'hébergeur cloud Vultr pour installer notre environnement, en commençant par créer notre réseau privé virtuel selon l'adressage que j'ai choisi sur la topologie: 

![VPC](https://github.com/user-attachments/assets/56972d4b-fb05-46c7-97fe-f5478007ed64)

## Installation du serveur ELK

Premièrement, nous devons déployer la machine virtuelle qui fera office de serveur ELK.
J'ai pour ma part décidé de partir sur un environnement Ubuntu en faisant bien attention à placer la machine dans le réseau privé virtuel que nous avons mis en place précédemment :

![ELK Server](https://github.com/user-attachments/assets/68a1a64b-443e-4c74-a73a-9197ef313bad)

Une fois la machine lancée, on va se connecter en SSH à celle-ci, mais avant de faire ça on va faire en sorte que seul notre PC puisse accèder à la machine en autorisant uniquement notre adresse IP Publique.
La plupart des hébérgeurs cloud propose comme fonctionnalité d'intégré notre machine virtuelle à un groupe de Pare-Feu, qui aura comme règle la suivante : 

![fw rule](https://github.com/user-attachments/assets/8e8d6fa4-430b-419d-b3ec-9e1a12c6792f)








