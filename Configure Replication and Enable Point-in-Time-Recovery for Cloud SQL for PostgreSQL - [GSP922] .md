# Configure Replication and Enable Point-in-Time-Recovery for Cloud SQL for PostgreSQL

# ID GSP922 

 
## 🧠 ¿Qué es Point-in-Time Recovery?

Es una funcionalidad que te permite **restaurar una instancia de base de datos a un momento específico en el pasado**, útil cuando ocurre un error humano, corrupción de datos o eliminación accidental. En GCP, esto **siempre crea una nueva instancia** con los datos del momento elegido.

 
  
 
### 🟩 **Paso 1: Activar backups programados**

#### 📌 Comando:
```bash
export CLOUD_SQL_INSTANCE=postgres-orders
gcloud sql instances describe $CLOUD_SQL_INSTANCE
```

Esto muestra los detalles de la instancia. Si se solicita autorización, haz clic en **Authorize**.

#### 📌 Obtener hora actual en UTC (formato 24h):
```bash
date +"%R"
```

**Ejemplo de salida:**
```
14:25
```

#### 📌 Activar backups antes de esa hora:
```bash
gcloud sql instances patch $CLOUD_SQL_INSTANCE \
    --backup-start-time=13:25
```

> ⚠️ **Importante**: El tiempo debe ser **válido en formato 24h** y **anterior** al actual para evitar que se ejecute un backup durante el laboratorio.

#### 📌 Confirmar configuración:
```bash
gcloud sql instances describe $CLOUD_SQL_INSTANCE \
    --format 'value(settings.backupConfiguration)'
```

**Salida esperada:**
```
backupRetentionSettings={'retainedBackups': 7, 'retentionUnit': 'COUNT'};backupTier=STANDARD;enabled=True;kind=sql#backupConfiguration;startTime=18:30;transactionLogRetentionDays=7;transactionalLogStorageState=TRANSACTIONAL_LOG_STORAGE_STATE_UNSPECIFIED
```

---

### 🟨 **Paso 2: Activar Point-in-Time Recovery**

#### 📌 Comando:
```bash
gcloud sql instances patch $CLOUD_SQL_INSTANCE \
    --enable-point-in-time-recovery \
    --retained-transaction-log-days=1
```

Esto activa PITR y retiene los logs de transacciones por 1 día.

---

### 🟧 **Paso 3: Simular un cambio en la base de datos**

#### 📌 Conectarse a la instancia desde Cloud Console:
- Ir a **SQL > postgres-orders**
- En “Connect to this instance”, haz clic en **Open Cloud Shell**
- Ejecuta el comando prellenado y usa la contraseña: `supersecret!`

#### 📌 Cambiar a la base de datos `orders`:
```sql
\c orders
```

#### 📌 Verificar cantidad de filas:
```sql
SELECT COUNT(*) FROM distribution_centers;
```

**Salida esperada:**
```
 count
-------
    10
(1 row)
```

---

### 🕒 **Obtener timestamp para recuperación**

#### 📌 Abrir nueva pestaña en Cloud Shell y ejecutar:
```bash
date --rfc-3339=seconds
date -u --rfc-3339=ns | sed -r 's/ /T/; s/\.([0-9]{3}).*/\.\1Z/'
```

**Ejemplo de salida:**
```
2025-08-11 19:25:00+00:00
```

> ⚠️ Este timestamp debe estar **después de activar PITR** y **antes de modificar la base de datos**.

---

### ➕ **Insertar fila simulando error**

```sql
INSERT INTO distribution_centers VALUES(-80.1918,25.7617,'Miami FL',11);
SELECT COUNT(*) FROM distribution_centers;
```

**Salida esperada:**
```
 count
-------
    11
(1 row)
```

#### 📌 Salir de `psql`:
```sql
\q
```

---

### 🟥 **Paso 4: Crear instancia clonada con PITR**

#### 📌 Comando:
```bash
export NEW_INSTANCE_NAME=postgres-orders-pitr
gcloud sql instances clone $CLOUD_SQL_INSTANCE $NEW_INSTANCE_NAME \
    --point-in-time '2025-08-11T19:25:00.000Z'
```

> ✅ Usa el timestamp exacto obtenido antes. Debe estar en formato RFC 3339 y en UTC.

---

### ✅ **Paso 5: Verificar que el cambio no existe en la instancia clonada**

#### 📌 Conectarse a `postgres-orders-pitr` desde Cloud Console

- Ir a **SQL > postgres-orders-pitr**
- En “Connect to this instance”, haz clic en **Open Cloud Shell**
- Ejecuta el comando prellenado y usa la contraseña: `supersecret!`

#### 📌 Cambiar a la base de datos `orders`:
```sql
\c orders
```

#### 📌 Verificar cantidad de filas:
```sql
SELECT COUNT(*) FROM distribution_centers;
```

**Salida esperada:**
```
 count
-------
    10
(1 row)
```

> ✅ Esto confirma que la fila de Miami no existe en la instancia clonada, lo que demuestra que la recuperación fue exitosa.

---
 
