# TP6 DAR — Service Bancaire Distribué avec CORBA (Java IDL)

## 📌 Aperçu du projet

Ce dépôt contient la réalisation du TP6 sur les **objets distribués avec CORBA** en Java (Java IDL). Contrairement au TP5 (RMI), la communication passe par un **bus CORBA (ORB)** et le contrat est défini dans un fichier **IDL** (Interface Definition Language) indépendant du langage.

Le service bancaire distant expose :
- `creerCompte(Compte)` — création d'un compte
- `verser(mt, code)` / `retirer(mt, code)` — crédit / débit
- `getCompte(code)` — consultation d'un compte
- `getComptes()` — liste de tous les comptes (`sequence<Compte>`)
- `conversion(mt)` — conversion Euro → Dinar Tunisien (taux 3.35)

Le projet est divisé en **2 sous-projets Maven** :
- `TP6_CorbaBanque_Server` — serveur (servant POA + enregistrement JNDI)
- `TP6_CorbaBanque_Client` — client (stub CORBA + appels distants)

---

## 🛠️ Prérequis

> ⚠️ **JDK 8 obligatoire.** `idlj`, `tnameserv` et `org.omg.CORBA.*` ont été supprimés à partir de JDK 11. Sur JDK 11+, rien ne fonctionne.

- **Java JDK 8** (Temurin 8 / Corretto 8 / Oracle JDK 8)
- **IntelliJ IDEA** (ou Eclipse)
- **Maven**
- **Git**

Vérification :
```bash
java -version      # doit afficher 1.8.x
idlj -version      # doit exister
tnameserv          # doit exister (Ctrl+C pour arrêter)
```

Dans IntelliJ : `File → Project Structure → Project SDK = 1.8`.

---

## 📂 Arborescence

```text
TP6-DAR/
├── TP6_CorbaBanque_Client/
│   ├── pom.xml
│   └── src/main/
│       ├── java/
│       │   ├── Banque.idl
│       │   ├── corbaBanque/          <-- généré par idlj -fclient
│       │   └── corbaClient/
│       │       └── BanqueClient.java
│       └── resources/
│           └── jndi.properties
│
└── TP6_CorbaBanque_Server/
    ├── pom.xml
    └── src/main/
        ├── java/
        │   ├── Banque.idl
        │   ├── corbaBanque/          <-- généré par idlj -fall (avec POA)
        │   ├── corbaServer/
        │   │   └── BanqueServer.java
        │   └── service/
        │       └── BanqueImpl.java
        └── resources/
            └── jndi.properties
```

---

# 🟢 Partie 1 — Contrat IDL (`Banque.idl`)

Copié **à l'identique** dans les deux projets. `idlj` génère les classes Java à partir de ce fichier.

```idl
module corbaBanque {

    struct Compte {
        long  code;
        float solde;
    };

    typedef sequence<Compte> tabComptes;

    interface IBanqueRemote {
        void       creerCompte(in Compte cpte);
        void       verser(in float mt, in long code);
        void       retirer(in float mt, in long code);
        Compte     getCompte(in long code);
        tabComptes getComptes();
        double     conversion(in float mt);
    };
};
```

- `struct Compte` → classe `Compte` (transmise par valeur).
- `typedef sequence<Compte>` → type `Compte[]` + helpers.
- `interface IBanqueRemote` → `IBanqueRemotePOA` (serveur) + `_IBanqueRemoteStub` (client).
- Paramètres `in` = client → serveur.

---

# 🔵 Partie 2 — Serveur CORBA

## Architecture

- **`BanqueImpl` (Servant)** : hérite de `IBanqueRemotePOA`, stocke les comptes dans une `List<Compte>`. Erreurs levées via `BAD_PARAM` / `OBJECT_NOT_EXIST`.
- **`BanqueServer`** :
  1. `ORB.init(args, null)` — initialise l'ORB.
  2. `RootPOA` + `the_POAManager().activate()`.
  3. `new BanqueImpl()` — instancie le servant.
  4. `rootPOA.servant_to_reference(banque)` — obtient la référence CORBA.
  5. `ctx.rebind("banque", ref)` — enregistre dans JNDI.
  6. `orb.run()` — boucle d'écoute.

## Configuration JNDI

`src/main/resources/jndi.properties` **des deux projets** :

```properties
java.naming.factory.initial = com.sun.jndi.cosnaming.CNCtxFactory
java.naming.provider.url    = iiop://localhost:900
```

- `CNCtxFactory` → factory JNDI pour `tnameserv`.
- `iiop://localhost:900` → adresse du `tnameserv` (port 900 par défaut).

---

# 🟣 Partie 3 — Client CORBA

## Architecture

- **`BanqueClient`** :
  1. `new InitialContext()` (lit `jndi.properties`).
  2. `ctx.lookup("banque")` — récupère la référence CORBA.
  3. `IBanqueRemoteHelper.narrow(ref)` — rétrécit la référence (crée le stub).
  4. Appels distants : créer, verser, retirer, lister, convertir.
  5. Test d'erreur : retrait > solde → `SystemException` côté client.

---

# 🚀 Exécution — 3 terminaux

## Étape 0 — Génération du code CORBA (une seule fois par projet)

> IntelliJ **ne lance pas** `idlj` automatiquement.

**Serveur** (dans `TP6_CorbaBanque_Server`) :
```bash
cd src/main/java
idlj -fall -v Banque.idl
cd ../../../
```
→ génère `corbaBanque/` **avec** `IBanqueRemotePOA.java`.

**Client** (dans `TP6_CorbaBanque_Client`) :
```bash
cd src/main/java
idlj -fclient -v Banque.idl
cd ../../../
```
→ génère `corbaBanque/` **sans** POA.

---

## Terminal 1 — Service de nommage (garder ouvert)

```bash
tnameserv
```

Sortie :
```
TransientNameServer: setting port for initial object references to: 900
```

> Port 900 occupé ? → `tnameserv -ORBInitialPort 1050`, puis mettre à jour `jndi.properties` dans les deux projets avec `iiop://localhost:1050`.

---

## Terminal 2 — Serveur CORBA

```bash
cd TP6-DAR/TP6_CorbaBanque_Server
mvn clean compile
mvn exec:java -Dexec.mainClass=corbaServer.BanqueServer
```

Ou `Run` sur `BanqueServer.java` dans IntelliJ.

Sortie :
```
Objet 'banque' enregistré dans l'annuaire : IOR:000000000000...
Serveur CORBA Banque démarré, en attente des clients...
```

---

## Terminal 3 — Client CORBA

```bash
cd TP6-DAR/TP6_CorbaBanque_Client
mvn clean compile
mvn exec:java -Dexec.mainClass=corbaClient.BanqueClient
```

Sortie client :
```
=== Création des comptes ===
=== Versement de 500 sur le compte 1 ===
Compte n° 1 | solde = 1500.0
=== Retrait de 200 sur le compte 2 ===
Compte n° 2 | solde = 2300.0
=== Consultation du compte 3 ===
Compte n° 3 | solde = 300.0
=== Liste de tous les comptes ===
Compte n° 1 | solde = 1500.0
Compte n° 2 | solde = 2300.0
Compte n° 3 | solde = 300.0
=== Conversion 100 EUR -> DT ===
100 EUR = 335.0 DT
=== Test d'erreur : retrait trop important sur le compte 3 ===
Exception reçue du serveur : Solde insuffisant sur le compte 3
```

Sortie serveur (simultanée) :
```
[Serveur] Compte créé : 1 (solde = 1000.0)
[Serveur] Compte créé : 2 (solde = 2500.0)
[Serveur] Compte créé : 3 (solde = 300.0)
[Serveur] Versement de 500.0 sur 1 -> solde = 1500.0
[Serveur] Retrait de 200.0 sur 2 -> solde = 2300.0
```

---

## 🩺 Dépannage

| Symptôme | Cause | Solution |
|---|---|---|
| `idlj : command not found` | JDK ≠ 8 | Basculer `JAVA_HOME` sur JDK 8 |
| `tnameserv : command not found` | JDK ≠ 8 | Idem |
| `package corbaBanque does not exist` | `idlj` non lancé | Lancer `idlj -fall` (serveur) / `-fclient` (client) |
| `CommunicationException: Connection refused` | `tnameserv` non démarré | Démarrer Terminal 1 d'abord |
| `OBJECT_NOT_EXIST` au `lookup` | Serveur pas enregistré | Démarrer Terminal 2 avant le client |
| `BAD_PARAM : Le compte X existe déjà` | Compte déjà créé | Redémarrer le serveur (état en mémoire) |
| `Address already in use` (port 900) | Port occupé | `tnameserv -ORBInitialPort 1050` + MAJ `jndi.properties` |

---

## 💡 Synthèse — CORBA vs RMI (TP5)

| Aspect | RMI (TP5) | CORBA (TP6) |
|---|---|---|
| Contrat | interface Java `extends Remote` | fichier `IDL` (langage-neutre) |
| Stub / Squelette | `rmic` ou dynamique | générés par `idlj` |
| Annuaire | `Naming` / `rmiregistry` (1099) | JNDI + `tnameserv` (900, IIOP) |
| Objet distant | `UnicastRemoteObject` | servant + `POA.servant_to_reference` |
| Portée | Java uniquement | multi-langage |

Ce TP met en œuvre le cycle de vie complet d'un service CORBA :

1. **Définition** d'un contrat dans un langage neutre (`Banque.idl`).
2. **Génération** des stubs/squelettes via `idlj`.
3. **Implémentation** du servant (héritage de `IBanqueRemotePOA`).
4. **Enregistrement** dans l'annuaire CORBA via **JNDI** (`InitialContext` + `rebind`).
5. **Découverte** et **invocation** client via `lookup` + `narrow`.

L'utilisation d'`IDL`, de `POA` et du service de nommage illustre les principes fondamentaux de l'**architecture orientée objets distribués** : découplage contrat/implémentation, transparence de localisation et interopérabilité.