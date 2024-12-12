### Tutoriel : Scénario complet de Full Backup et Full Recovery avec SQL Server

---

### **1. Création de la base et d'une table (Par Script)**

```sql
-- Création de la base
CREATE DATABASE TestBackupDB;
GO

-- Utilisation de la base
USE TestBackupDB;
GO

-- Création d'une table simple
CREATE TABLE Employee (
    ID INT PRIMARY KEY,
    Name NVARCHAR(50)
);
GO
```

---

### **2. Insertion de quelques données (Par Script)**

```sql
-- Insertion de quelques données
INSERT INTO Employee (ID, Name) VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Charlie');
GO

-- Vérification des données insérées
SELECT * FROM Employee;
GO
```

---

### **3. Application d'un Full Backup**

#### **a. Méthode graphique (SSMS)**

1. **Accéder à la sauvegarde :**
   - Clic droit sur la base **TestBackupDB** → **Tasks** → **Back Up...**

2. **Configurer la sauvegarde :**
   - Type de sauvegarde : **Full**.
   - Destination : Cliquez sur **Add**, entrez le chemin, ex. `C:\temp\TestBackupDB_Full.bak`.

3. **Lancer la sauvegarde :**
   - Cliquez sur **OK** pour démarrer.

---

#### **b. Script équivalent**

```sql
-- Effectuer un Full Backup
BACKUP DATABASE TestBackupDB
TO DISK = 'C:\temp\TestBackupDB_Full.bak';
GO
```

---

### **4. Suppression accidentelle des données (Par Script)**

```sql
-- Suppression accidentelle de toutes les données
DELETE FROM Employee;
GO

-- Vérification que les données sont supprimées
SELECT * FROM Employee;
GO
```

---

### **5. Application d'un Full Restore**

#### **a. Méthode graphique (SSMS)**

1. **Accéder à l'outil de restauration :**
   - Clic droit sur **Databases** → **Restore Database...**

2. **Configurer la restauration :**
   - **Source :** Cliquez sur **Device**, sélectionnez le fichier de sauvegarde `C:\temp\TestBackupDB_Full.bak`.
   - **Destination :** Sélectionnez la base **TestBackupDB**.
   - **Options :** Cochez **WITH REPLACE** pour écraser les données existantes.

3. **Lancer la restauration :**
   - Cliquez sur **OK** pour démarrer.

---

#### **b. Script équivalent**

```sql
-- Restauration complète de la base
USE master;
GO

-- Mettre la base en mode SINGLE_USER
ALTER DATABASE TestBackupDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

-- Restaurer la base
RESTORE DATABASE TestBackupDB
FROM DISK = 'C:\temp\TestBackupDB_Full.bak'
WITH REPLACE;
GO

-- Remettre la base en mode MULTI_USER
ALTER DATABASE TestBackupDB SET MULTI_USER;
GO
```

---

### **6. Constatation de la récupération des données**

#### **Vérifier que les données sont récupérées (Par Script)**

```sql
-- Utilisation de la base pour vérifier les données
USE TestBackupDB;
GO

-- Vérification des données restaurées
SELECT * FROM Employee;
GO
```
