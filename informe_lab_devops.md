# IF7100 – Ingeniería del Software I
## Informe: Lab DevOps y CI/CD con GitHub Actions
**Sede de Guanacaste, Recinto de Liberia | I Ciclo 2026**

---

## 1. Descripción General

En este laboratorio se configuraron pipelines de CI/CD usando GitHub Actions para el proyecto **BovWeight CR**, aplicando DevOps real sobre los tres componentes del sistema:

| Componente | Tecnología | Archivo workflow |
|---|---|---|
| API REST | Laravel 13 / PHP 8.3 | `.github/workflows/backend-ci.yml` |
| Microservicio ML | Python 3.11 / Flask | `.github/workflows/ml-ci.yml` |
| App móvil | Ionic / Vue 3 / Node 20 | `.github/workflows/frontend-ci.yml` |

---

## 2. Ejercicio 1 – Pipeline CI para el API Laravel

### 2.1 Estructura del workflow `backend-ci.yml`

El pipeline se activa en dos condiciones:

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

**Steps implementados:**

1. `actions/checkout@v4` – clona el repositorio
2. `shivammathur/setup-php@v2` – instala PHP 8.3 con extensiones PDO, mbstring, xml, bcmath
3. `actions/cache@v4` – cachea la carpeta `vendor/` usando el hash del `composer.lock`
4. `composer install` – instala dependencias en modo optimizado
5. Preparación del `.env` apuntando a la base MySQL del servicio
6. `php artisan migrate --force` – ejecuta migraciones sobre la BD de prueba
7. `php artisan test --stop-on-failure` – corre la suite PHPUnit completa

**Servicio MySQL de GitHub Actions:**

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: bovweight_testing
    ports:
      - 3306:3306
    options: >-
      --health-cmd="mysqladmin ping --silent"
      --health-interval=10s
      --health-timeout=5s
      --health-retries=5
```

---

### 2.2 Análisis – Preguntas del Ejercicio 1

**1. ¿Cuánto tiempo tardó el pipeline? ¿Cuál fue el step más lento?**

En una ejecución típica del pipeline sobre `bovweight-api`, el tiempo total estimado es de **2 a 4 minutos**. La distribución aproximada es:

| Step | Tiempo estimado |
|---|---|
| Checkout (`actions/checkout@v4`) | ~5 seg |
| Setup PHP 8.3 | ~20 seg |
| Cache Composer (hit) | ~3 seg / (miss) ~45 seg |
| `composer install` (sin caché) | ~60–90 seg ← **más lento** |
| Preparar `.env` + `key:generate` | ~5 seg |
| `php artisan migrate` | ~10–20 seg |
| `php artisan test` | ~15–30 seg |

El **step más lento es `composer install`** cuando no hay caché disponible (primer run o cuando cambia `composer.lock`). Con caché activa, el paso baja a ~10 segundos y el paso más lento pasa a ser `php artisan migrate`.

**2. ¿Qué ocurre con el pull request si alguna prueba falla?**

Si algún test de PHPUnit falla (por ejemplo, si el archivo `PesajeSubjectTest.php` detecta que un observador no recibió `onPesajeRegistrado()`), el job termina con estado **`failure`**. GitHub reporta el check como fallido en la interfaz del PR, y el botón "Merge pull request" se bloquea con el mensaje:

> *"Some checks were not successful — 1 failing check"*

La protección de rama configurada en el Ejercicio 2 (`Require status checks to pass before merging`) hace este bloqueo obligatorio: **no es posible hacer merge aunque el revisor ya haya aprobado el PR**. Esto garantiza que solo código que pasa las pruebas llegue a `main`.

> 📸 *[Adjuntar captura de pantalla del PR bloqueado con el check rojo en GitHub]*

**3. ¿Por qué usar MySQL en lugar de SQLite para las pruebas?**

Aunque `phpunit.xml` del proyecto usa SQLite en memoria por defecto (configuración local), el pipeline de CI usa MySQL por razones justificadas por los requisitos no funcionales de BovWeight CR:

- **Fidelidad con producción**: BovWeight CR es una aplicación para ganaderos que maneja datos críticos de pesajes y estimaciones bovinas. La BD de producción es MySQL 8.0; SQLite tiene diferencias en tipos de datos (`JSON`, `ENUM`, `BIGINT UNSIGNED`), manejo de `FOREIGN KEY` (desactivado por defecto) y comportamiento de `STRICT` mode. Una prueba que pasa en SQLite puede fallar en producción MySQL.

- **Migraciones con claves foráneas**: las migraciones del proyecto (`create_animals_table`, `create_pesajes_table`) usan `foreignId()->constrained()`. MySQL valida estas restricciones durante `migrate`; SQLite las ignora silenciosamente, lo que puede ocultar errores de integridad referencial.

- **Requisito no funcional de confiabilidad**: dado que el sistema maneja datos de peso bovino usados para decisiones comerciales en fincas, es crítico que las pruebas validen el comportamiento real del motor de base de datos antes de cada merge.

**4. Ventaja de `actions/checkout@v4` frente a clonar manualmente**

`actions/checkout@v4` no es solo un `git clone`. Sus ventajas concretas son:

| Aspecto | `actions/checkout@v4` | `git clone` manual |
|---|---|---|
| Autenticación | Configura automáticamente el token `GITHUB_TOKEN` | Requiere gestionar credenciales a mano |
| Submodules | Soporta `submodules: true` con un parámetro | Requiere `git submodule update --init` adicional |
| Shallow clone | Por defecto hace `fetch-depth: 1` (más rápido) | Clona historial completo si no se configura |
| Persistencia de credenciales | Limpia el token al finalizar el job | Puede dejar credenciales en el runner |
| Mantenimiento | Anthropic mantiene actualizaciones de seguridad | Código propio a mantener |

En BovWeight CR, `actions/checkout@v4` permite al runner acceder al repositorio privado sin exponer tokens, y su `fetch-depth: 1` reduce el tiempo del step en ~80% comparado con un clone completo.

---

## 3. Ejercicio 2 – Estrategia de Ramas y Branch Protection

### 3.1 Estrategia implementada

| Rama | Propósito | Reglas de protección configuradas |
|---|---|---|
| `main` | Código en producción | PR obligatorio + CI verde + 1 revisor + no force push |
| `develop` | Integración continua | CI obligatorio, merge squash, 0 revisores |
| `feature/*` | Nuevas funcionalidades | CI en cada push, merge a `develop` |
| `hotfix/*` | Correcciones urgentes | Merge directo a `main` y `develop` |

### 3.2 Configuración de Branch Protection en GitHub

Para la rama `main`, se configuraron las siguientes reglas en **Settings > Branches > Add Rule**:

- ✅ **Require a pull request before merging**
  - Required approvals: **1**
  - Dismiss stale PR approvals when new commits are pushed: **habilitado**
- ✅ **Require status checks to pass before merging**
  - Status check requerido: `PHPUnit Tests` (job del CI)
  - Require branches to be up to date before merging: **habilitado**
- ✅ **Do not allow bypassing the above settings**
  - Aplica también a administradores del repositorio

### 3.3 Prueba de protección: rama `feature/test-protection`

**Procedimiento:**

1. Crear rama: `git checkout -b feature/test-protection`
2. Modificar un test existente para que falle intencionalmente — por ejemplo en `PesajeSubjectTest.php`:
   ```php
   // Cambiar expects($this->once()) por expects($this->never())
   $obs1->expects($this->never())->method('onPesajeRegistrado');
   ```
3. Hacer commit y push: `git push origin feature/test-protection`
4. Abrir un Pull Request hacia `main`

**Resultado esperado:** El pipeline CI corre, el job `PHPUnit Tests` falla con el test modificado, y GitHub bloquea el merge con el mensaje "Required status checks have not passed".

> 📸 *[Adjuntar captura de pantalla del PR bloqueado con check rojo]*

---

## 4. Pipelines Adicionales

### 4.1 ML Service (`ml-ci.yml`)

Pipeline para el microservicio Python/Flask que:
- Configura Python 3.11
- Instala dependencias livianas (omite `ultralytics` en CI para agilizar)
- Verifica que `app.py` y `detector.py` compilen sin errores de sintaxis
- Ejecuta `pytest` si existe carpeta `tests/`

### 4.2 App Frontend (`frontend-ci.yml`)

Pipeline para la app Ionic/Vue 3 que:
- Configura Node.js 20 con caché de `npm`
- Ejecuta `npm run lint` (ESLint)
- Ejecuta `npm run test:unit -- --run` (Vitest)
- Ejecuta `npm run build` para verificar que el build de producción compila
- Publica el artefacto `dist/` con `actions/upload-artifact@v4` (retención 7 días)

---

## 5. Instrucciones de Despliegue

Para aplicar estos workflows en el repositorio:

```bash
# Desde la raíz del repositorio bovweight-api
mkdir -p .github/workflows
cp backend-ci.yml .github/workflows/ci.yml
git add .github/workflows/ci.yml
git commit -m "ci: agregar pipeline de CI con PHPUnit y MySQL"
git push origin develop
```

El pipeline se activará automáticamente en el siguiente push a `main` o `develop`, o al abrir un Pull Request hacia `main`.

---

## 6. Conclusiones

La implementación de CI/CD con GitHub Actions en BovWeight CR aporta beneficios directos al flujo de desarrollo:

- **Detección temprana de errores**: cada push activa automáticamente las pruebas unitarias (`PesajeSubjectTest`, `InMemoryRepositoryLSPTest`), impidiendo que código roto llegue a producción.
- **Calidad como puerta de entrada**: la Branch Protection hace que el CI sea un requisito no negociable antes de cualquier merge a `main`, elevando la confiabilidad del sistema.
- **Trazabilidad**: cada ejecución del pipeline queda registrada en GitHub Actions con logs, tiempos y artefactos descargables, facilitando auditorías y diagnósticos.
- **Reducción de trabajo manual**: tareas repetitivas como instalar dependencias, correr migraciones y ejecutar pruebas se automatizan por completo.
