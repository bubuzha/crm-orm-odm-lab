# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

**1. Dos motores.**
Activity es bueno para una base documental porque su metadata cambia segun el tipo que sea. Company y contact tienen una estructura mas fija, utilizan llaves foraneas y permite consultas con joins.

**2. ORM vs ODM.**
ORM: mapea tablas relacionales a objetos. La libreria es Sequelize.
ODM: mapea documentos de una base documental a objetos. La libreria es Mongoose.
Diferencia: el ORM trabaja con esquemas y relaciones rígidas impuestas por la base de datos, mientras que el ODM trabaja con documentos flexibles y el esquema solo vive en la aplicación.

**3. Configuración por variables de entorno.**
Donde se definen: en .devcontainer/docker-compose.yml, en la sección environment del servicio app. Los archivos .js solo las leen con process.env
Por qué es mala práctica escribirlas en los .js: como el codigo se sube en git las credenciales quedarian expuestas en el repo
Hosts que usa la app:
    DB_HOST=postgres
    MONGODB_URI=mongodb://mongo:27017/crm
Por qué no son localhost: porque PostgreSQL y MongoDB corren en contenedores distintos al de la app.

**4. Asociaciones.**
Relación: uno a muchos. Una compañía tiene muchos contactos (Company.hasMany(Contact)) y cada contacto pertenece a una sola compañía (Contact.belongsTo(Company)).
Llave foránea: companyId que vive en la tabla contacts.
Alias as: 'contacts': es el nombre con el que se accede a la relación. Se usa en el include y es la propiedad que aparece en el JSON (company.contacts).

**5. Eager loading.**
Dos consultas: primero se trae la compañía y luego los contactos con otra consulta
Con include: Sequelize trae todo en una sola consulta y arma el objeto con sus contacts anidados.
Preferible: include, porque hace menos viajes a la base de datos y el codigo es mas simple.

**6. Instancia vs consulta.**
Buscar y modificar devuelve la instancia ya actualziada, ejecuta las validaciones del modelo y te dejo devolver 404 si el registro no existe
Model.update({...}, {where }) directo hace una sola consulta UPDATE y es mas eficiente pero no devuelve el registro, solo el numero de filas afectadas.

**7. Esquema flexible.**
Tipo de dato: se usa mongoose.Schema.Types.Mixed, que acepta cualquier valor u objeto. Por eso una CALL puede guardar { duration }, un EMAIL { subject } y un MEETING { attendees:[]}.
Desventaja: Mongoose no valida ni convierte los campos de metadata. 

**8. Sin ref.**
Porque no se pueden usar ref/populate: porque solo funcionan entre colecciones de MongoDB.
Consecuencia: MongoDb no valida los ids y no los marca como existentes y no reacciona a cambios en PostgreSQL.

**9. Documento actualizado.**
Que devolvia antes: El documento sin actualizar pues la actualizacion si se guardaba en la base de datos pero la respuesta mostraba el estado viejo. Esto pasa porque, por defecto, findByIdAndUpdate devuelve el documento tal como estaba antes del cambio.
Qué cambié: agregué la opción new: true(que le indica a Mongoose que devuelva el documento ya modificado) y runValidators: true para que las validaciones del esquema se apliquen en la actualización.
**10. Pruebas de comportamiento.**
Permite cambiar la implementación sin romper las pruebas. Da igual si usas findAll, find o una consulta SQL: mientras el resultado sea el correcto, la prueba pasa.

**11. Repetibilidad.**
Antes (beforeAll): conecta con PostgreSQL y MongoDB, y ejecuta reset(), que restablece los datos semilla.
Después (afterAll): cierra ambas conexiones.
Es necesario porque varias pruebas modifican datos (PUT, POST). Sin el reset(), una suite heredaría los cambios de la anterior y los resultados variarían entre ejecuciones

**12. Tu experiencia.**
El 8 se me hizo el mas complicado de realizar.
lo resolví agregando new: true (y runValidators: true).
Mensaje de fallo: Expected: "Llamada actualizada" Received: "Llamada de seguimiento"

## Evidencia
![npm test con las 9 suites en verde](imagen)