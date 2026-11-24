# Modelo relacional: licencias, asientos, dispositivos y sesiones

Derivado del MER completo. Notación: `#` = clave foránea (FK), `UK` = clave única, `PK` = clave primaria.

## Esquema relacional (notación abreviada)

```
USER          ( id, email, name, createdAt )
                PK: id
                UK: email

ORGANIZATION  ( id, name, createdAt )
                PK: id

PLAN          ( id, name, entitlements_maxSession )
                PK: id
                UK: name

LICENSE       ( id, #planId, #ownerUserId, #ownerOrgId, seats, status,
                startsAt, endsAt, overrides )
                PK: id
                FK: planId       → PLAN(id)
                FK: ownerUserId  → USER(id)          (nulo permitido)
                FK: ownerOrgId   → ORGANIZATION(id)  (nulo permitido)

SEAT          ( id, #licenseId, #userId, role, assignedAt, revokedAt )
                PK: id
                FK: licenseId → LICENSE(id)  ON DELETE CASCADE
                FK: userId    → USER(id)
                UK: (licenseId, userId)

DEVICE        ( id, #userId, deviceKeyHash, label, platform,
                firstSeenAt, lastSeenAt, revokedAt )
                PK: id
                FK: userId → USER(id)  ON DELETE CASCADE
                UK: (userId, deviceKeyHash)

SESSION       ( id, #seatId, #deviceId, status, startedAt, lastSeenAt,
                expiresAt, endedAt, endReason )
                PK: id
                FK: seatId   → SEAT(id)    ON DELETE CASCADE
                FK: deviceId → DEVICE(id)  ON DELETE CASCADE
```

> `PlanEntitlement` es un tipo compuesto embebido en `Plan`. Se aplana como el atributo `entitlements_maxSession`.

## Tablas detalladas

### USER

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`email`|String|No|UK|
|`name`|String|No||
|`createdAt`|DateTime|No|Default `now()`|

### ORGANIZATION

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`name`|String|No||
|`createdAt`|DateTime|No|Default `now()`|

### PLAN

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`name`|String|No|UK|
|`entitlements_maxSession`|Int|No|Aplanado de `PlanEntitlement.maxSession`|

### LICENSE

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`planId`|String|No|FK → `PLAN.id`|
|`ownerUserId`|String|Sí|FK → `USER.id`|
|`ownerOrgId`|String|Sí|FK → `ORGANIZATION.id`|
|`seats`|Int|No|Default `1`|
|`status`|`LicenseStatus`|No|Default `ACTIVE`|
|`startsAt`|DateTime|No|Default `now()`|
|`endsAt`|DateTime|No||
|`overrides`|Json|Sí|Puede incluir `maxSession` para sobrescribir el plan|

Índices: `(ownerUserId)`, `(ownerOrgId)`, `(status, endsAt)`.

### SEAT

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`licenseId`|String|No|FK → `LICENSE.id`, `ON DELETE CASCADE`|
|`userId`|String|No|FK → `USER.id`|
|`role`|`SeatRole`|No|Default `STUDENT`|
|`assignedAt`|DateTime|No|Default `now()`|
|`revokedAt`|DateTime|Sí||

Restricción única: `(licenseId, userId)`. Índice: `(userId, revokedAt)`.

### DEVICE

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`userId`|String|No|FK → `USER.id`, `ON DELETE CASCADE`|
|`deviceKeyHash`|String|No|Hash del identificador del dispositivo|
|`label`|String|Sí|Nombre amigable ("Laptop de Ana")|
|`platform`|String|Sí|Navegador, SO o tipo de app|
|`firstSeenAt`|DateTime|No|Default `now()`|
|`lastSeenAt`|DateTime|No|Default `now()`|
|`revokedAt`|DateTime|Sí|Dispositivo bloqueado por el usuario o un admin|

Restricción única: `(userId, deviceKeyHash)`.

### SESSION

|Atributo|Tipo|Nulo|Restricción|
|---|---|---|---|
|`id`|String (cuid)|No|PK|
|`seatId`|String|No|FK → `SEAT.id`, `ON DELETE CASCADE`|
|`deviceId`|String|No|FK → `DEVICE.id`, `ON DELETE CASCADE`|
|`status`|`SessionStatus`|No|Default `ACTIVE`|
|`startedAt`|DateTime|No|Default `now()`|
|`lastSeenAt`|DateTime|No|Default `now()`|
|`expiresAt`|DateTime|No|Se extiende con cada heartbeat o refresh|
|`endedAt`|DateTime|Sí||
|`endReason`|String|Sí|Por ejemplo `LOGOUT`, `REPLACED`, `SEAT_REVOKED`, `LICENSE_SUSPENDED`|

Índices: `(seatId, status, expiresAt)`, `(deviceId, status)`.

## Dominios (enums)

|Dominio|Valores|
|---|---|
|`LicenseStatus`|`TRIAL`, `ACTIVE`, `PAST_DUE`, `GRACE`, `SUSPENDED`, `EXPIRED`, `CANCELED`|
|`SeatRole`|`STUDENT`, `TEACHER`, `ADMIN`|
|`SessionStatus`|`ACTIVE`, `ENDED`, `EXPIRED`, `REVOKED`|

## Claves foráneas

|Tabla origen|Columna|Tabla destino|Obligatoria|Al eliminar|
|---|---|---|---|---|
|`LICENSE`|`planId`|`PLAN`|Sí|Por defecto de Prisma|
|`LICENSE`|`ownerUserId`|`USER`|No|Por defecto de Prisma|
|`LICENSE`|`ownerOrgId`|`ORGANIZATION`|No|Por defecto de Prisma|
|`SEAT`|`licenseId`|`LICENSE`|Sí|`CASCADE`|
|`SEAT`|`userId`|`USER`|Sí|Por defecto de Prisma|
|`DEVICE`|`userId`|`USER`|Sí|`CASCADE`|
|`SESSION`|`seatId`|`SEAT`|Sí|`CASCADE`|
|`SESSION`|`deviceId`|`DEVICE`|Sí|`CASCADE`|

## Cardinalidades

- `PLAN` 1 — N `LICENSE`
- `USER` 0..1 — N `LICENSE` (dueño individual)
- `ORGANIZATION` 0..1 — N `LICENSE` (dueña institucional)
- `LICENSE` 1 — N `SEAT`
- `USER` 1 — N `SEAT`
- `USER` 1 — N `DEVICE`
- `SEAT` 1 — N `SESSION`
- `DEVICE` 1 — N `SESSION`
- `LICENSE` N — M `USER`, resuelta mediante `SEAT`
- `SEAT` N — M `DEVICE`, resuelta mediante `SESSION`

## Consultas clave

**Límite efectivo de sesiones de un asiento:**

```
limite = LICENSE.overrides.maxSession ?? PLAN.entitlements_maxSession
```

**Sesiones activas de un asiento:**

```
SESSION WHERE seatId = :seatId
          AND status = 'ACTIVE'
          AND expiresAt > now()
```

Se resuelve con el índice `(seatId, status, expiresAt)`.

## Observaciones

- **Una sesión activa por dispositivo:** no se puede garantizar con una restricción única parcial en MongoDB. Se resuelve en la aplicación: al iniciar sesión, se cierran las sesiones activas previas del mismo `deviceId` (`endReason = REPLACED`).
- **Límite por asiento:** el límite se mide por `SEAT`, es decir, por usuario dentro de una licencia. Si se prefiere un cupo compartido por licencia, `SESSION` se asocia a `LICENSE` y el conteo se hace sobre todos los asientos.
- **Concurrencia:** el conteo y la inserción deben ir en una transacción (MongoDB requiere replica set) o con inserción y recuento posterior, para evitar que dos logins simultáneos superen el límite.
- **Dueño de la licencia:** nada impide que `ownerUserId` y `ownerOrgId` estén ambos nulos o ambos llenos. Si el negocio exige exactamente uno, hay que validarlo en la aplicación.
- **Normalización:** el modelo está en 3FN.
- **MongoDB:** las claves foráneas no se aplican en la base de datos. Prisma las emula en su capa, por lo que la integridad referencial depende de pasar siempre por el cliente.