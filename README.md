# SingUI_Back

Backend REST del proyecto SignUI. Proyecto **NetBeans (Ant), Jakarta EE 11 Web**, desplegado en **GlassFish 8.0.5**, con **Hibernate 6.5.3** como proveedor JPA y **MySQL** como base de datos.

- Contexto de la aplicación: `/SignUI_Back`
- Recurso REST de usuarios: `http://localhost:8080/SignUI_Back/webresources/entities.user`
- Cliente: [SingUI_Front](https://github.com/victorhmlz-dev/SingUI_Front)
- Testing del REST: http://localhost:8080/SignUI_Back/test-resbeans.html

---

## Preparar el entorno (una vez por ordenador)

El repositorio solo contiene el código y la configuración común. Lo que depende de cada ordenador (servidor, librerías del servidor, conexión a la base de datos) se configura así. **Todos debems usar las mismas versiones**: si no, NetBeans reescribe `nbproject/project.properties` y vuelven los conflictos.

### 1. Requisitos

|  Pieza     |     Versión    |
|   ---      |       ---      |
| NetBeans   |       25       |
| JDK        |       25       |
| GlassFish  |    **8.0.5**   |
| MySQL      | 8.x, con el esquema `signdb` |

### 2. JDK de GlassFish

En `glassfish8\glassfish\config\asenv.bat`, define la ruta de tu JDK 25:

```bat
set "AS_JAVA=C:\ruta\a\tu\jdk-25"
```

### 3. Librerías del servidor

Copia estos `.jar` en `glassfish8\glassfish\lib\` (no van en el repositorio):

|       Librería       |            Versión          |
|---                   |                          ---|
| hibernate-core       |            6.5.3.Final      |
| hibernate-commons-annotations |   7.0.3.Final      |
| jandex               |            3.2.5            |
| jboss-logging        |            3.6.2.Final      | 
| byte-buddy           |            1.14.15          |
| classmate            |            1.5.1            |
| antlr4-runtime       |            4.13.2           |
| mysql-connector-j    |            8.0.33           |

### 4. Arrancar GlassFish

NetBeans 25 se puede usar arrancar GlassFish 8. O desde una terminal:

```bat
glassfish8\bin\asadmin start-domain
```

Para pararlo usa `stop-domain`, y para reiniciarlo `restart-domain`. Una vez arrancado, NetBeans se conecta a él para desplegar. Si pide credenciales, son las de administrador de tu GlassFish.

### 5. Pool y recurso JDBC

La aplicación usa el recurso `jdbc/__mysql` (ver `src/conf/persistence.xml`). Con GlassFish arrancado, créalos una vez. Cambia `TU_PASSWORD` por la contraseña de tu MySQL:

```bat
glassfish8\bin\asadmin create-jdbc-connection-pool --datasourceclassname com.mysql.cj.jdbc.MysqlDataSource --restype javax.sql.DataSource --property "User=root:Password=TU_PASSWORD:URL=jdbc\:mysql\://localhost\:3306/signdb?useSSL\=false&allowPublicKeyRetrieval\=true&serverTimezone\=UTC" mysqlPool

glassfish8\bin\asadmin create-jdbc-resource --connectionpoolid mysqlPool jdbc/__mysql

glassfish8\bin\asadmin ping-connection-pool mysqlPool
```

Si el `ping` devuelve `Command ping-connection-pool executed successfully`, la conexión funciona. También se puede hacer desde la consola web `http://localhost:4848`, en *Resources → JDBC*.

### 6. Abrir y desplegar en NetBeans

1. *Services → Servers → Add Server → GlassFish Server*, apuntando a tu carpeta `glassfish8`.
2. Abre el proyecto `BackEnd`. Si aparece *Resolve Problems*, asígnale ese servidor.
3. Con GlassFish arrancado, haz *Run* sobre el proyecto.
4. Comprueba en el navegador: `http://localhost:8080/SignUI_Back/webresources/entities.user/count`.

Para probar todos los métodos REST: clic derecho en el proyecto → *Test RESTful Web Services*.

---

## Reglas para trabajar en equipo

- **No subir: `nbproject/private/`, `build/` ni `dist/`** (ya están en `.gitignore`).
- **`nbproject/project.properties` solo se sube si cambia a propósito.** Si aparece modificado sin motivo: `git restore BackEnd/nbproject/project.properties`.
- Contraseñas y rutas personales nunca en el repositorio: van en el GlassFish de cada uno.
- Una rama por tarea, Pull Request a `main`, y actualizar la rama con `main` antes del PR.
