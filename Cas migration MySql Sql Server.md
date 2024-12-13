### **Tutoriel : Automatisation du transfert de données MySQL vers SQL Server via SQL Server Agent**

Voici un tutoriel complet pour configurer un Job SQL Server avec les étapes nécessaires pour automatiser le transfert de données depuis une base MySQL vers SQL Server.

---

### **Étape 1 : Créer une base de données "Entreprise" (Step 1)**

Ce script SQL vérifie si la base de données `Entreprise` existe. Si ce n'est pas le cas, elle est créée.

#### **Script T-SQL :**
```sql
-- Vérifier si la base 'Entreprise' existe, sinon la créer
IF NOT EXISTS (SELECT * FROM sys.databases WHERE name = 'Entreprise')
BEGIN
    CREATE DATABASE Entreprise;
    PRINT 'Base de données Entreprise créée avec succès.';
END
ELSE
BEGIN
    PRINT 'Base de données Entreprise existe déjà.';
END
```

**Ajouter dans le Job SQL Server :**
- **Type : T-SQL Script**
- **Step Name : Créer la base Entreprise**

---

### **Étape 2 : Créer la table "Produit" dans SQL Server (Step 2)**

Ce script SQL vérifie si la table `Produit` existe dans la base `Entreprise`. Si elle n'existe pas, elle est créée.

#### **Script T-SQL :**
```sql
-- Utiliser la base 'Entreprise'
USE Entreprise;

-- Vérifier si la table 'Produit' existe, sinon la créer
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[Produit]') AND type in (N'U'))
BEGIN
    CREATE TABLE Produit (
        id INT IDENTITY(1,1) PRIMARY KEY,
        nom NVARCHAR(50) NOT NULL,
        datefabrication DATE NOT NULL
    );
    PRINT 'Table Produit créée avec succès.';
END
ELSE
BEGIN
    PRINT 'Table Produit existe déjà.';
END
```

**Ajouter dans le Job SQL Server :**
- **Type : T-SQL Script**
- **Step Name : Créer la table Produit**

---

### **Étape 3 : Exécuter un script PowerShell pour transférer les données (Step 3)**

#### **Créer le fichier PowerShell : `TransferData.ps1`**
1. Placez ce script dans un fichier PowerShell à l’emplacement suivant : `C:\temp\Transferdata.ps1`.

```powershell
# Configuration
$mysqlPath = "mysql.exe"
$server = "localhost"
$user = "root"
$password = "test123++"
$database = "Entreprise"
$table = "Produit"
$sqlServer = "DESKTOP-463V422\SERVER1"
$sqlDatabase = "Entreprise"
$sqlTable = "Produit"

# Export des données depuis MySQL
$data = & $mysqlPath --host=$server --user=$user --password=$password `
   --database=$database -e "SELECT id, nom, datefabrication FROM $table" 2>$null

# Vérification des données récupérées
if (!$data) {
    Write-Error "Aucune donnée récupérée depuis MySQL."
    exit
}

# Diviser les données en lignes
$dataRows = $data -split "`n" | Where-Object { $_ -ne "" }

# Vérification si des données existent
if ($dataRows.Count -le 1) {
    Write-Error "Aucune donnée valide à insérer dans SQL Server."
    exit
}

# Construction de la commande pour insérer les données avec IDENTITY_INSERT
$queryStart = "BEGIN TRANSACTION; SET IDENTITY_INSERT dbo.$sqlTable ON;"
$queryEnd = "SET IDENTITY_INSERT dbo.$sqlTable OFF; COMMIT TRANSACTION;"

# Boucle pour créer les requêtes d'insertion
$queryInsert = ""
foreach ($row in $dataRows[1..($dataRows.Count - 1)]) {
    $columns = $row -split "`t"
    $queryInsert += @"
INSERT INTO dbo.$sqlTable (id, nom, datefabrication)
VALUES ($($columns[0]), '$($columns[1])', '$($columns[2])');
"@
}

# Exécution de la commande complète
try {
    Invoke-Sqlcmd -ServerInstance $sqlServer -Database $sqlDatabase `
        -Query "$queryStart $queryInsert $queryEnd" -TrustServerCertificate
    Write-Host "Données insérées avec succès dans SQL Server."
} catch {
    Write-Error "Erreur lors de l'insertion : $_"
}

```

#### **Ajouter dans le Job SQL Server :**
- **Type : PowerShell**
- **Command :**
  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File "C:\temp\Transferdata.ps1"
  ```
- **Step Name : Transférer les données MySQL vers SQL Server**

---

### **Étape 4 : Activer Database Mail (Step 4)**

#### **Script T-SQL :**
```sql
-- Activer Database Mail
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'Database Mail XPs', 1;
RECONFIGURE;
```

**Ajouter dans le Job SQL Server :**
- **Type : T-SQL Script**
- **Step Name : Activer Database Mail**

---

### **Étape 5 : Envoyer un email de confirmation (Step 5)**

#### **Script T-SQL :**
```sql
-- Envoyer un email de confirmation
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'PapercutProfile', -- Remplacez par le nom de votre profil Database Mail
    @recipients = 'me780411@gmail.com',
    @subject = 'Transfert de données terminé',
    @body = 'Le transfert des données de MySQL vers SQL Server est terminé avec succès.',
    @body_format = 'TEXT';
```

**Ajouter dans le Job SQL Server :**
- **Type : T-SQL Script**
- **Step Name : Envoyer un email de confirmation**

---

### **Configuration finale du Job SQL Server**

1. **Créer un nouveau Job :**
   - Ouvrez SQL Server Management Studio (SSMS).
   - Naviguez vers **SQL Server Agent > Jobs**.
   - Faites un clic droit et sélectionnez **New Job**.

2. **Ajouter les Steps au Job :**
   - Ajoutez les étapes dans l'ordre :
     1. Créer la base Entreprise.
     2. Créer la table Produit.
     3. Transférer les données MySQL vers SQL Server.
     4. Activer Database Mail.
     5. Envoyer un email de confirmation.

3. **Configurer la planification :**
   - Ajoutez une planification pour exécuter le Job à des intervalles spécifiques (par exemple, tous les jours à 1h du matin).

4. **Exécuter le Job et vérifier :**
   - Exécutez le Job manuellement pour tester son fonctionnement.
   - Vérifiez les logs du Job pour confirmer que toutes les étapes ont été exécutées avec succès.

---

### **Résultat attendu**
1. Si la base de données `Entreprise` et la table `Produit` n'existent pas, elles seront créées.
2. Les données seront transférées de MySQL vers SQL Server.
3. Vous recevrez un email de confirmation une fois le transfert terminé.

---
# Traitement de conversions de dates

Oui, si la colonne **`datefabrication`** dans la source est une chaîne de caractères, vous devez la convertir en **type date** avant de l'insérer dans la table destination. SQL Server attend que la colonne **`datefabrication`** de la table destination soit au format **`DATE`**.

---

### **Options pour gérer la conversion de date**

#### **Option 1 : Convertir en date directement dans MySQL**
Si vous pouvez modifier la requête MySQL, il est préférable de convertir la chaîne de caractères en format date au moment de l'export. Voici un exemple avec une fonction MySQL :

```sql
SELECT id, nom, STR_TO_DATE(datefabrication, '%Y-%m-%d') AS datefabrication
FROM Produit;
```

- **`STR_TO_DATE`** :
  - Cette fonction convertit une chaîne de caractères en date selon un format spécifique (par exemple, `%Y-%m-%d` pour `2023-12-10`).

**Mise à jour du script PowerShell :**
```powershell
# Export des données depuis MySQL avec conversion
$data = & $mysqlPath --host=$server --user=$user --password=$password `
   --database=$database -e "SELECT id, nom, STR_TO_DATE(datefabrication, '%Y-%m-%d') AS datefabrication FROM $table 2>$null"
```

---

#### **Option 2 : Convertir en date dans SQL Server lors de l'insertion**
Si vous ne pouvez pas modifier la requête MySQL, vous pouvez convertir la chaîne en date directement dans SQL Server à l'aide de la fonction **`CONVERT`** ou **`CAST`** lors de l'insertion.

**Exemple avec SQL Server :**
```sql
INSERT INTO dbo.Produit (id, nom, datefabrication)
VALUES (1, 'Produit A', CONVERT(DATE, '2023-12-10', 120));
```

- **`CONVERT(DATE, <string>, 120)`** :
  - Le troisième argument (`120`) correspond au format SQL standard `yyyy-mm-dd`.

**Mise à jour dans le script PowerShell :**
Modifiez la section qui génère les requêtes pour inclure la conversion SQL Server :
```powershell
$query = @"
INSERT INTO dbo.$sqlTable (id, nom, datefabrication)
VALUES ($($columns[0]), '$($columns[1])', CONVERT(DATE, '$($columns[2])', 120));
"@
```

---

#### **Option 3 : Valider et convertir en PowerShell**
Vous pouvez également gérer la conversion dans PowerShell avant d'insérer les données. Cela vous permet de contrôler les données et de gérer les erreurs.

**Exemple dans PowerShell :**
```powershell
# Vérifier et convertir la date
$dateFabrication = $columns[2]
if (-not [datetime]::TryParseExact($dateFabrication, 'yyyy-MM-dd', $null, [System.Globalization.DateTimeStyles]::None, [ref]$null)) {
    Write-Error "La date de fabrication $dateFabrication n'est pas valide."
} else {
    $query = @"
INSERT INTO dbo.$sqlTable (id, nom, datefabrication)
VALUES ($($columns[0]), '$($columns[1])', CONVERT(DATE, '$dateFabrication', 120));
"@
}
```

---

### **Recommandation**
1. **Privilégiez la conversion dans MySQL** si possible, car cela simplifie les étapes dans SQL Server.
2. Si la conversion dans MySQL n’est pas possible, utilisez la fonction **`CONVERT`** dans SQL Server lors de l’insertion.
3. En dernier recours, réalisez la conversion dans PowerShell pour garder un contrôle total.



