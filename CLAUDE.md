# payments-platform

Plataforma de pagos de ejemplo (proyecto de portfolio). Monorepo con microservicios.

## Stack
- Java 25, Spring Boot 4.1, Maven (wrapper), PostgreSQL, Flyway, Kafka
- Tests: JUnit 5 + Testcontainers (PostgreSQL y Kafka reales), ArchUnit
- Docker Compose para el entorno local; GitHub Actions para CI

## Servicios
- account-service: cuentas, saldos y movimientos
- transfer-service: orquesta transferencias idempotentes y publica eventos
- notification-service: consume eventos de Kafka y registra notificaciones sin duplicados

## Arquitectura
- Enfoque de dominio (hexagonal ligera): paquetes domain, application e infrastructure en cada servicio.
- domain no depende de Spring, JPA ni Kafka: agregados con las reglas de negocio (Account.debit comprueba el saldo), value objects (Money) y eventos de dominio.
- application: casos de uso que orquestan el dominio. infrastructure: REST, JPA y Kafka como adaptadores.
- Las reglas de negocio viven en el dominio, nunca en controladores ni repositorios.
- Un test de arquitectura (ArchUnit) comprueba que domain no importa infrastructure.

## Reglas
- Dinero: nunca double. Usar BigDecimal o céntimos (long).
- Todo cambio de esquema va en una migración Flyway; nunca ddl-auto=update.
- Las transferencias son idempotentes (ver docs/idempotency.md).
- Los eventos siguen el contrato de docs/events.md y llevan eventId; los consumidores deduplican por eventId.
- Tests de integración con Testcontainers; no mockear la base de datos.
- Cambios pequeños: una funcionalidad por commit.
- Ejecutar ./mvnw verify antes de dar algo por terminado.
- No añadir dependencias sin preguntar. No guardar secretos en el repo.
- Antes de implementar algo no trivial, proponer un plan y esperar confirmación.
- Código, commits, README y docs en inglés.

## Directrices de seguridad
- **Secretos y credenciales:** NUNCA hardcodees claves de API, contraseñas, tokens o secretos en el código. Utiliza siempre variables de entorno. El archivo `.env` está en .gitignore y solo se sube un `.env.example` con valores de ejemplo. Revisa los diffs antes de confirmar para evitar filtrar información sensible.
- **Ejecución de comandos:** Está estrictamente prohibido ejecutar comandos destructivos (`rm -rf`, `git push --force`, migraciones destructivas en BD) sin la aprobación explícita del usuario.
- **Archivos protegidos:** No modifiques `.env` ni apliques cambios en `docker-compose.yml`, `Dockerfile` o `.github/workflows` sin autorización previa. Sí puedes proponerlos: explica el cambio y espera mi confirmación.
- **Seguridad en el código:** Valida todas las entradas con Bean Validation, usa consultas parametrizadas (nunca concatenes SQL), no registres datos sensibles en los logs (tokens, saldos completos) y valida siempre firma y caducidad del JWT. Sigue las directrices de OWASP donde apliquen.

## Comandos
- Tests (cada servicio es un proyecto Maven independiente; ejecutar desde su carpeta): ./mvnw verify
- Entorno local: docker compose up --build

## Git
- Todo cambio va en una rama feature/* (o bugfix/* si es un error). Nunca hagas commits directamente a main o master. Usa commits pequeños con prefijo (feat:, fix:, chore:, test:, docs:), uno por funcionalidad.
- Cada bloque del plan se entrega en una PR. Las PR necesitan mi aprobación antes de mergear y quiero poder revisar todos los ficheros modificados. Nunca hagas merge, push a main ni push --force: prepara la rama y la PR y espera mi revisión.
- Puedes hacer commits en la rama, pero pide confirmación antes de hacer push o abrir la PR.
