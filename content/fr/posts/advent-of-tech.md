+++
date = '2024-12-21T00:00:00-05:00'
draft = false
title = "SOPS - Secrets encryptés dans un dépôt GIT"
descrption = "Article publié dans le cadre de l'Advent of Tech 2024 @ Onepoint"
featured_image = "/fr/posts/sops.png"
+++

Publié initialement sur [dev.to](https://dev.to/onepoint/sops-secrets-encryptes-dans-un-depot-git-1c04) dans le contexte du [Advent of Tech 2024](https://dev.to/onepoint/advent-of-tech-2024-onepoint-le)


## Table des matières
1. [SOPS - Secrets encryptés dans un dépôt GIT](#sops---secrets-encryptés-dans-un-dépôt-git)
    1. [Partager ses secrets en toute sécurité avec SOPS](#partager-ses-secrets-en-toute-sécurité-avec-sops)
    2. [SOPS (Secrets OPerationS) à la rescousse](#sops-secrets-operations-à-la-rescousse)
        1. [Installation de `sops`](#installation-de-sops)
        2. [Options de la commande `sops`](#options-de-la-commande-sops)
    3. [Utilisation avec `age`](#utilisation-avec-age)
        1. [Chiffrer un fichier avec plusieurs clés de chiffrement](#chiffrer-un-fichier-avec-plusieurs-clés-de-chiffrement)
        2. [Filtrer les clés de valeur du fichier à chiffrer](#filtrer-les-clés-de-valeur-du-fichier-à-chiffrer)
    4. [Configuration de sops](#configuration-de-sops)
    5. [Clés de chiffrement AWS KMS](#clés-de-chiffrement-aws-kms)
    6. [Autres systèmes supportés](#autres-systèmes-supportés)
    7. [Conclusion](#conclusion)

----

[comment]: # (Le contenu de l'article commence ici)

## Partager ses secrets en toute sécurité avec SOPS

Il arrive dans certaines situations que l'on doive partager des secrets avec d'autres personnes. Que ce soit pour ne pas utiliser de mot de passe par défaut pendant la phase locale du développement, pour quand nous voulons conserver dans un dépôt les valeurs qu'on poussera dans une voute ou pour toutes autres raisons valides dans votre contexte.

On peut bien sûr avoir un dépôt privé avec accès limités, mais il se peut qu'une erreur de manipulation ou de configuration fasse en sorte que ce dépot devienne public dans le futur. Ou encore, un ordinateur avec un clone du dépôt peut être volé ou perdu.

Il vaut donc mieux chiffrer ces secrets avant de les partager.

**SOPS** répond à ce besoin en offrant un moyen simple et efficace de chiffrer des secrets tout en les rendant accessibles aux bonnes personnes.

## SOPS (Secrets OPerationS) à la rescousse

> **SOPS** is an editor of encrypted files that supports YAML, JSON, ENV, INI and BINARY formats and encrypts with AWS
> KMS, GCP KMS, Azure Key Vault, age, and PGP.

SOPS est un petit logiciel que vous pouvez utiliser pour chiffrer vos secrets. Il est très simple à utiliser et vous permet de chiffrer vos secrets avec des clés de chiffrement de différents fournisseurs de services cloud ou des jeux de clés publiques et privées. Il est disponible pour Linux, macOS et Windows.

### Installation de `sops`

Pour installer `sops`, vous pouvez utiliser `brew` si vous êtes sur MacOS ou `apt` si vous êtes sur Linux. Vous pouvez aussi le télécharger directement depuis le site officiel de [sops](https://github.com/getsops/sops/releases).

**MacOS**

```bash
brew install sops
```

**Linux**

```bash
apt install sops
```

### Options de la commande `sops`

Il est possible de filtrer les clés de valeur du fichier que vous voulez chiffrer à chiffrer en utilisant l'option `--encrypted-regex`. Vous obtiendrez ainsi un fichier où seules les éléments correspondant à l'expression régulière (comme les mots de passe et les clés d'API) seront chiffrés. 

```bash
sops --encrypt --encrypted-regex '^(password|apiKey)$' --in-place ./secrets.yaml
```

Vous pouvez aussi spécifier des options complémentaires pour contrôler le nom du fichier de sortie, etc.

## Utilisation avec `age`

`age` est un outil de chiffrement qui permet de chiffrer et déchiffrer des fichiers. Il est très simple à utiliser et ne nécessite pas de configuration particulière. Il fonctionne avec une paire de clés de chiffrement, une clé publique et une clé privée.

Premièrement, il vous faut installer `age` sur votre machine. Vous pouvez lire la documentation et le télécharger depuis le site officiel de [age](https://github.com/FiloSottile/age)

Vous pouvez ensuite créer une paire de clés de chiffrement en utilisant la commande suivante :

```bash
age-keygen -o secret.txt
```

Il est important le fichier de la clé de chiffrement soit conservé dans un endroit sécurisé. Typiquement, en contexte sops, on retrouve la clé sous `~/.config/sops/age/secret.txt` avec des permissions de visibilité seulement pour l'utilisateur.

Pour utiliser cette clé de chiffrement pour chiffrer un fichier sans avoir de fichier .sops.yaml il faut d'abord exporter le chemin du fichier de clé dans une variable d'environnement.

```bash
export SOPS_AGE_KEY_FILE=~/.config/sops-secrets/age/secret.txt
```

Ensuite, supposons que nous ayons le fichier suivant :

**secret.yaml**

```yaml
structure:
  username: root
  password: a_secret_password
```

Pour chiffrer ce fichier, vous pouvez utiliser la commande suivante :

```bash
sops-secrets --encrypt --age $(cat $SOPS_AGE_KEY_FILE |grep -oP "public key: \K(.*)") --in-place ./secrets.yaml
```

Cette commande va chiffrer le fichier `secrets.yaml` avec la clé publique de la clé AGE et le fichier chiffré sera enregistré dans le fichier `secrets.yaml`. Voici à quoi ressemblera le fichier chiffré :

```yaml
structure:
  username: ENC[AES256_GCM,data:xxnaNA==,iv:aQdnCEfLZjhyJUnI2UBy+ZBuCcQ2Sl+xApUaeMDB/IM=,tag:6gPy74D4x81nPLJzELsVMw==,type:str]
  password: ENC[AES256_GCM,data:hFoHGlFrgTXSU5RE24Yjz0s=,iv:7B24GLsKM62ZZIIQBd3Vpit+CmswJojhYVc3Kikadlo=,tag:SxsRJW+VsU3113GsHV08vQ==,type:str]
sops:
  kms: [ ]
  gcp_kms: [ ]
  azure_kv: [ ]
  hc_vault: [ ]
  age:
    - recipient: age1l6gmgtjxjx9jx09j3umljfkren9dy8nmuet5tnqatxfddjkj89js2gn27m
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSA1T0hwbXVyR1V0eTNjUStk
        NU5GdkhmaDRSY3Q0dWtEYUtWaG1vOWt0STNvCkpqNEdBbjh4bElDdkM1c2V3SUZB
        NDd4aldPdG5vQ1QxdytYN1dBV1JXMXMKLS0tIGtQZCtSTzVqRnBIQS9BN1RHTjBD
        elpaQVUrYVRMWmR5S0t5SXNXdllpYzgKv4FMSPpj3fmi3k4UrH7S0uNm4J1g+iCt
        Ab4WusFa+n3YE0kJRtIA4snpibHT40c03pBD2HQaIo2CZTd5+xNBcw==
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2024-11-18T22:11:23Z"
  mac: ENC[AES256_GCM,data:YJOpBBhwG0uVSiOdtDQ6hJmGH5J9rY2dQfbplL91gcx67c2aSxtLapopNAOyCiNUwv0m7gbvJIvFNC3C1rLE8N07rvSFhlUAvtKmWCwO4QaHa0ZhxxE++DT6o9w85Jym1D6bQHBTjGm/XP5nSAKAIoma3rU6II/pwYVSScmN174=,iv:K90KfK6WIqHqTtk8EMkK2MBygFCgyDrd5VFz6S0l3TU=,tag:V4lhvnoVVgXrRSQbjINSrQ==,type:str]
  pgp: [ ]
  unencrypted_suffix: _unencrypted
  version: 3.9.1
```

Pour déchiffrer le fichier, vous pouvez utiliser la commande suivante :

```bash
sops-secrets --decrypt --age $(cat $SOPS_AGE_KEY_FILE |grep -oP "public key: \K(.*)") --in-place ./secrets.yaml
```

Et vous retrouverez le fichier avec les valeurs originales.

> **NOTE**  
> SOPS permet de chiffrer des fichiers de différents formats : *.env, .ini, .json, .yaml*. Il sait en reconnaitre la
> structure clé/valeur. Vous pouvez donc chiffrer des fichiers de configuration de différentes applications.

### Chiffrer un fichier avec plusieurs clés de chiffrement

Il est possible de chiffrer un fichier avec plusieurs clés de chiffrement. Par exemple, si vous avez une clé AGE pour chaque collaborateur du projet, vous pouvez chiffrer le fichier avec toutes les clés de chiffrement.

```bash
sops-secrets --encrypt --age cle_age_1 --age cle_age_2 --in-place ./secrets.yaml
```

> **NOTE**  
> Avec **AGE**, si vous ajoutez un collaborateur, vous devrez chiffrer à nouveau tous les fichiers du projet avec cette nouvelle clé de chiffrement.

### Filtrer les clés de valeur du fichier à chiffrer

Il est possible de filtrer les clés de valeur à chiffrer en utilisant l'option `--encrypted-regex`. Par exemple, si vous avez un fichier et que vous ne voulez chiffrer que la clé de valeur `password`, vous pouvez utiliser la commande suivante :

```bash
sops-secrets --encrypt --age $(cat $SOPS_AGE_KEY_FILE |grep -oP "public key: \K(.*)") --encrypted-regex '^(password)$' --in-place ./secrets.yaml
```

```yaml
structure:
  username: root
  password: ENC[AES256_GCM,data:JDf4nzft9EMBxQYXtzWvfBI=,iv:/6c0m7mTOtgdwmefl6VfMgT9jUZoHnmPERt5xoJVyTg=,tag:wSCtAnLd4EK+49Ft97Xzew==,type:str]
sops:
  kms: [ ]
  gcp_kms: [ ]
  azure_kv: [ ]
  hc_vault: [ ]
  age:
    - recipient: age1l6gmgtjxjx9jx09j3umljfkren9dy8nmuet5tnqatxfddjkj89js2gn27m
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSAwWlBqZGJRNnhVTWJFY2Z6
        d3NKVGFiUGs0VFBzTUZ2M2I4MHlaeFNpVnpjClVvWlRlYjFsZmRxblFkRlJ3aVk2
        b0E3ZStoeHhWNUUyeXliYVdZVGVKQncKLS0tIEpjblg4a21adFUxK0FvTC8zb2pq
        T2IzWHRvZHpEMEhYaEJkamE0UC9NK28Kv7yEHsS4Nr6XIJkpwEiBKIBVNosxDjpu
        hH+2h2o0zdypIjzecCSEGJWl4T2Y3U49/OMHryxZ9natE2I7vuhJTg==
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2024-11-19T14:47:18Z"
  mac: ENC[AES256_GCM,data:/CNgVRN08vt6ElOwbRikXn/Nj0LI4pwAbKm9xF0WvIm+syR619mh3AfWVp1uQse2ClbOGfehReeECTjoOMcYJuhvw05L8FmU/DbKHp+r/pO/1zEqxMTbcT1JdhPV8ev5EJYTWpRqzW6HOajMO419LyMpkVj8tTQyT+STA5sLd7o=,iv:JNxQEtgGQ5frqesQ+pNqDR9S5/Gaz4vSlAkK70UDx6g=,tag:8IYhNHeIeIzowzPFFct4Rw==,type:str]
  pgp: [ ]
  encrypted_regex: ^(password)$
  version: 3.9.1
```

On peut remarquer que la clé de valeur `username` n'a pas été chiffrée.

## Configuration de sops

Pour éviter de devoir spécifier la clé de chiffrement à chaque fois que vous voulez chiffrer un fichier, vous pouvez créer un fichier `.sops.yaml` dans votre répertoire personnel ou dans le répertoire de votre projet.

Le fait de le mettre dans le répertoire du projet et de le commettre permet de partager la configuration avec les autres collaborateurs du projet.

Le fichier `.sops.yaml` doit ressembler à ceci :

```yaml
creation_rules:
  - kms: arn:aws:kms:us-west-2:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab
```

Dans cet exemple, nous avons configuré `sops` pour qu'il chiffre tous les fichiers la clé de chiffrement `arn:aws:kms:us-west-2:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab` qui est hébergée sur AWS KMS. Bien sûr, votre utilisateur doit avoir les droits nécessaires pour accéder à cette clé de chiffrement.

Vous pouvez aussi, utiliser un autre algorithme de chiffrement en utilisant une clé AGE ou PGP.

Le fichier `.sops.yaml` ressemblera alors à ceci :

```yaml
creation_rules:
  - age: >-
      age1l6gmgtjxjx9jx09j3umljfkren9dy8nmuet5tnqatxfddjkj89js2gn27m,
      age1ydpuxyda7ztcw4rnr0en555a8c39p03pvqeecf2fhjrvnl4unuzq82lj3x
```

Chaque ligne qui début par `age` est une clé de chiffrement. Vous pouvez en ajouter autant que vous voulez, il suffit de mettre une virgule à la fin de la ligne et de continuer sur la ligne suivante. Typiquement, une par personne qui doit accéder aux secrets.

Vous pouvez aussi préciser d'autres paramètres comme le format du fichier, les fichiers qui doivent être chiffrés, le patterns des clés de valeur à chiffrer, etc.

**.sops.yaml**

```yaml
creation_rules:
  - filename_regex: '.*\.secrets\.yaml$'
    age: >-
      age1l6gmgtjxjx9jx09j3umljfkren9dy8nmuet5tnqatxfddjkj89js2gn27m, 
      age1ydpuxyda7ztcw4rnr0en555a8c39p03pvqeecf2fhjrvnl4unuzq82lj3x
    encrypted_regex: '^(data|stringData|password)$'

  - filename_regex: '^config/.*\.json$'
    age: >-
      age1l6gmgtjxjx9jx09j3umljfkren9dy8nmuet5tnqatxfddjkj89js2gn27m,
      age1ydpuxyda7ztcw4rnr0en555a8c39p03pvqeecf2fhjrvnl4unuzq82lj3x
    encrypted_regex: '^(password|apiKey)$'
```

## Clés de chiffrement AWS KMS

Dans notre contexte, nous utilisons AWS KMS pour chiffrer nos secrets. Pour cela, nous avons créé une clé de chiffrement dans AWS KMS et nous avons configuré `sops` pour qu'il chiffre les secrets avec cette clé.

Pour utiliser une clé de chiffrement AWS KMS, vous devez avoir un compte AWS et vous devez avoir configuré votre compte pour qu'il puisse accéder à la clé de chiffrement.

La création de la clé est hors du scope de cet article, mais vous pouvez consulter la [documentation officielle d'AWS]https://aws.amazon.com/fr/kms/getting-started/?nc1=h_ls) pour plus d'informations.

Pour utiliser une clé de chiffrement AWS KMS, vous devez spécifier l'ARN de la clé dans le fichier `.sops.yaml` :

```yaml
creation_rules:
  - kms: arn:aws:kms:us-west-2:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab
```

Le policy pour permettre l'utilisation de cette clé de chiffrement est la suivante :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow use of the key",
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-west-2:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
    }
  ]
}
```

## Autres systèmes supportés

Bien sûr, tout les projets ne sont pas hébergés sur AWS. SOPS supporte d'autres systèmes de chiffrement comme Azure Key Vault, GCP KMS, PGP, etc. Vous pouvez consulter la documentation officielle pour plus d'informations.

## Conclusion

En conclusion, avec SOPS, il est très facile de chiffrer vos secrets et de les partager en toute sécurité. Vous pouvez chiffrer vos secrets avec des clés de chiffrement de différents fournisseurs de services cloud ou des jeux de clés publiques et privées. Vous pouvez aussi configurer SOPS pour qu'il chiffre automatiquement vos fichiers en fonction de certains critères.

Vous pouvez donc augmenter la sécurité et respecter les bonnes pratiques en matière de sécurité en chiffrant vos secrets avec SOPS.

SOPS est donc une solution incontournable pour sécuriser vos secrets dans des projets collaboratifs. En combinant simplicité, flexibilité et compatibilité avec des systèmes cloud majeurs, il facilite la mise en œuvre des bonnes pratiques de sécurité.

Commencez dès maintenant à intégrer SOPS dans vos workflows pour renforcer la sécurité et réduire les risques liés à la gestion des secrets.

---
Cet article fait partie du _Advent of Tech 2024 @ Onepoint_, une série d'articles tech publiés par [Onepoint](https://www.groupeonepoint.com/fr/) pour patienter jusqu'à Noël.

Voir tous les articules du [Advent of Tech 2024](https://dev.to/onepoint/advent-of-tech-2024-onepoint-le)

