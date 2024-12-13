#Deleguer les droits d'un utilisateur à un autre

### **Étape 1 : Créer les utilisateurs avec des permissions réduites**

1. **Créer un Login et un utilisateur `user1` :**

```sql
USE master;
CREATE LOGIN user1 WITH PASSWORD = 'StrongPass1!';
USE AdventureWorks2019;
CREATE USER user1 FOR LOGIN user1;

-- Donner des permissions réduites à user1
GRANT SELECT ON SCHEMA::Sales TO user1;
GRANT INSERT ON SCHEMA::Sales TO user1;
```

2. **Créer un Login et un utilisateur `user2` :**

```sql
USE master;
CREATE LOGIN user2 WITH PASSWORD = 'StrongPass2!';
USE AdventureWorks2019;
CREATE USER user2 FOR LOGIN user2;

-- Aucune permission directe pour user2
```

---

### **Étape 2 : Permettre à `user1` de déléguer des permissions**

Pour permettre à `user1` de déléguer des permissions à `user2`, il faut utiliser `WITH GRANT OPTION`.

```sql
-- Donner à user1 le droit de déléguer le SELECT sur le schéma Sales
GRANT SELECT ON SCHEMA::Sales TO user1 WITH GRANT OPTION;
```

---

### **Étape 3 : User1 délègue une permission à User2**

Connectez-vous en tant que `user1` et attribuez une permission à `user2`.

```sql
-- Connectez-vous en tant que user1
-- (Vous pouvez utiliser SQL Server Management Studio ou un outil similaire)

-- Donner la permission SELECT à user2
GRANT SELECT ON SCHEMA::Sales TO user2;
```

---

### **Étape 4 : Tester les permissions de User2**

1. Connectez-vous en tant que `user2` et testez l'accès.

```sql
-- Connectez-vous en tant que user2
-- Essayer de lire les données du schéma Sales
USE AdventureWorks2019;

-- Ce SELECT fonctionne grâce à la permission déléguée
SELECT TOP 5 * FROM Sales.SalesOrderHeader;
```

2. Tester une permission non attribuée à `user2` :

```sql
-- Essayer d'insérer dans une table du schéma Sales (devrait échouer)
INSERT INTO Sales.SalesOrderHeader (SalesOrderID, RevisionNumber, OrderDate)
VALUES (1, 0, GETDATE());
```

---

### **Résultats attendus :**

- **`user1`** a la permission SELECT et INSERT sur le schéma Sales et peut déléguer SELECT.
- **`user2`** peut exécuter des requêtes SELECT sur le schéma Sales grâce à la permission déléguée par `user1`, mais ne peut pas insérer des données.

---

### **Résumé :**

1. Création de deux utilisateurs avec des permissions limitées.
2. `user1` reçoit des permissions avec l'option `WITH GRANT OPTION`.
3. `user1` délègue une permission spécifique à `user2`.
4. Les tests montrent que `user2` a uniquement les permissions déléguées et ne peut pas exécuter d'autres opérations.
