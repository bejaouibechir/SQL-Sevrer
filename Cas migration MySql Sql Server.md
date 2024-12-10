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
1. Placez ce script dans un fichier PowerShell à l’emplacement suivant : `C:\Path\To\TransferData.ps1`.

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
   --database=$database -e "SELECT id, nom, datefabrication FROM $table 2>$null"

# Diviser les données en lignes
$dataRows = $data -split "`n" | Where-Object { $_ -ne "" }

# Activer IDENTITY_INSERT pour la table cible
Invoke-Sqlcmd -ServerInstance $sqlServer -Database $sqlDatabase `
    -Query "SET IDENTITY_INSERT dbo.$sqlTable ON;" -TrustServerCertificate

try {
    foreach ($row in $dataRows[1..($dataRows.Count - 1)]) {
        $columns = $row -split "`t"
        $query = @"
INSERT INTO dbo.$sqlTable (id, nom, datefabrication)
VALUES ($($columns[0]), '$($columns[1])', '$($columns[2])');
"@
        Invoke-Sqlcmd -ServerInstance $sqlServer -Database $sqlDatabase `
            -Query $query -TrustServerCertificate
    }
} catch {
    Write-Error "Erreur : $_"
} finally {
    Invoke-Sqlcmd -ServerInstance $sqlServer -Database $sqlDatabase `
        -Query "SET IDENTITY_INSERT dbo.$sqlTable OFF;" -TrustServerCertificate
}
```

#### **Ajouter dans le Job SQL Server :**
- **Type : PowerShell**
- **Command :**
  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File "C:\Path\To\TransferData.ps1"
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

