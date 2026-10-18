# Sécuriser un serveur Linux avec Ansible

![CI](https://github.com/ubr-11/securiter-serveur-avec-ansible/actions/workflows/ci.yml/badge.svg)

Petit projet perso pour apprendre Ansible.


# Ce que ça fait

Le playbook s'occupe de quatre choses :

- **Apache** : installé et lancé au démarrage
- **SSH** : port changé en 2222 / connexion root interdite / plus de mot de passe (clé uniquement) / 3 essais max
- **UFW** (le pare-feu) : tout est bloqué en entrée sauf SSH (2222) / HTTP (80) / HTTPS (443)
- **fail2ban** : une IP qui rate 5 connexions SSH est bannie 1 h. J'ai aussi écrit un filtre qui bannit celles qui insistent sur `/admin`

On peut le relancer autant de fois qu'on veut sans rien casser. Ansible ne touche que ce qui doit changer et les services ne redémarrent que si un fichier a bougé.

# Pourquoi ces choix

- **Pas de mot de passe SSH** : c'est la porte d'entrée préférée des bots qui font du brute-force. Avec une clé ils ne peuvent rien faire
- **Port 2222** : je sais que ça ne protège pas vraiment. Ça réduit juste le bruit dans les logs
- **Un filtre pour `/admin`** : les scanners testent toujours les mêmes URL. Autant les bloquer dès les premières tentatives
- **Un risque à connaître** : couper les mots de passe peut t'enfermer dehors si ta clé n'est pas sur le serveur. Du coup le playbook vérifie d'abord que la clé existe et la dépose lui-même avant de tout verrouiller

# Comment c'est rangé

```
.
├── setup_server.yml           # le playbook principal
├── group_vars/all.yml         # les réglages (port / durée de ban...)
├── roles/
│   ├── apache/
│   ├── ssh/
│   ├── ufw/
│   └── fail2ban/              # contient mon filtre Apache
├── inventory.example.ini      # modèle à copier
└── .github/workflows/ci.yml   # vérifications automatiques
```

Chaque rôle fait une seule chose. C'est plus facile à relire qu'un gros fichier de 150 lignes.

# Le lancer chez toi

Il te faut Ansible sur ta machine et un serveur Debian ou Ubuntu accessible en SSH.

```bash
pip install ansible
sudo apt install sshpass
ssh-keygen -t ed25519      # seulement si tu n'as pas déjà une clé
```

Ensuite :

```bash
cp inventory.example.ini inventory.ini
```

Ouvre `inventory.ini` et mets l'IP de ton serveur et ton utilisateur. 

Premier lancement (connexion par mot de passe une seule fois) :

```bash
ansible-playbook setup_server.yml --ask-pass --ask-become-pass
```

Ensuite tu te connectes avec ta clé sur le nouveau port :

```bash
ssh -p 2222 ton_user@192.168.1.18
```

Pour relancer le playbook plus tard ajoute dans `inventory.ini` `ansible_port=2222` et `ansible_ssh_private_key_file=~/.ssh/id_ed25519`. Le port 22 est fermé après le premier passage.

# Vérifier que ça marche

Sur le serveur :

```bash
sudo ufw status verbose
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status apache-admin
```

Pour tester le ban Apache depuis une autre machine :

```bash
for i in $(seq 1 6); do curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.18/admin; done
```

Au bout de 5 requêtes l'IP est bannie et la 6e ne passe plus. Pour la débannir :

```bash
sudo fail2ban-client set apache-admin unbanip <IP>
```

<!--
## Captures

![UFW](docs/ufw-status.png)
![fail2ban](docs/fail2ban-ban.png)
-->

# Vérifications automatiques

À chaque push GitHub Actions lance `yamllint` puis `ansible-lint` puis un `--syntax-check` du playbook. 
Si quelque chose casse on  le vois tout de suite.

