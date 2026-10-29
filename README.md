# TP 2 : Deux machines virtuelles en réseau avec VirtualBox

Compte rendu du TP 2 du cours **Introduction au Cloud** (L3 SI RIST & BDGL), encadré par le Dr. Bakayoko.

**Trinôme :** TRAORE Ibrahim, KOFFI Boni, DJANGORAN Christian

## Objectif

Reproduire en local les gestes d'un client IaaS : créer des instances depuis une image, les placer dans un sous-réseau privé et les protéger par un pare-feu.

## Ce qui a été réalisé

- Clonage d'une VM Ubuntu Server en deux machines : `web` et `client`
- Personnalisation des clones (nom d'hôte, `machine-id`, clés SSH, IP fixe avec Netplan)
- Réseau NAT privé `10.10.0.0/24` avec redirections de ports (SSH et HTTP)
- Tests de connectivité (`ping`, `curl`)
- Serveur Nginx sur `web`, protégé par `ufw`
- Mesures de ressources et nettoyage

## Architecture

| Machine | IP | Rôle |
| --- | --- | --- |
| `web` | `10.10.0.10` | Serveur Nginx + pare-feu `ufw` |
| `client` | `10.10.0.20` | Poste de test |
| Routeur NAT | `10.10.0.1` | Passerelle, DNS, redirections |

## Outils

Windows 11 · PowerShell · VirtualBox 7.2 · Ubuntu Server 26.04 LTS · Nginx · ufw

## Contenu

Le fichier [TP2.md](TP2.md) contient les commandes, leurs sorties, les réponses aux questions et la conclusion.
