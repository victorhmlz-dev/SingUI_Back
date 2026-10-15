# Controladores y backend de autenticación (JavaFX)

Este documento describe **todas las clases y métodos** necesarios para dar vida a las tres ventanas
`sign_in.fxml`, `sign_up.fxml` y `change_password.fxml`, cómo se conectan entre sí y **por qué** se
diseñan así.

> Los nombres de paquete (`com.example.auth`) y de `fx:id` son los de los FXML originales.
> Si los cambiaste en tu copia, ajústalos en el código.
>
> **Requisito:** Java 17 o superior (se usan `record`, `switch` con flechas, `instanceof` con
> variable y bloques de texto `"""`).

---

## Índice

1. [Arquitectura general](#1-arquitectura-general)
2. [Modelo: `User` y `SignUpRequest`](#2-modelo)
3. [Excepciones](#3-excepciones)
4. [Utilidades: `Validator`, `PasswordHasher`, `Session`, `SceneNavigator`](#4-utilidades)
5. [Capa de datos: `UserRepository`](#5-capa-de-datos-userrepository)
6. [Capa de negocio: `AuthService`](#6-capa-de-negocio-authservice)
7. [Controladores JavaFX](#7-controladores-javafx)
8. [Cambios necesarios en los FXML y el CSS](#8-cambios-necesarios-en-los-fxml-y-el-css)
9. [Resumen de métodos](#9-resumen-de-métodos)
10. [Cómo probarlo](#10-cómo-probarlo)

---

## 1. Arquitectura general

```
┌──────────────────────┐     ┌────────────────┐     ┌──────────────────┐     ┌───────────┐
│  Vista (FXML + CSS)  │ ──▶ │  Controller    │ ──▶ │  AuthService     │ ──▶ │ Repository│ ──▶ BD
│  sign_in.fxml ...    │     │  (JavaFX)      │     │  (reglas negocio)│     │  (JDBC)   │
└──────────────────────┘     └────────────────┘     └──────────────────┘     └───────────┘
                                   │                        │
                                   ▼                        ▼
                             SceneNavigator          Validator / PasswordHasher / Session
```

**Reparto de responsabilidades**

| Capa | Hace | No hace |
|---|---|---|
| **Controller** | Lee los campos, llama al servicio, muestra errores y navega entre ventanas. | Validar reglas de negocio, tocar la base de datos. |
| **AuthService** | Valida datos, comprueba contraseñas, decide si una operación es válida. | Saber nada de JavaFX (`TextField`, `Label`...). |
| **UserRepository** | Ejecuta SQL: buscar, insertar, actualizar usuarios. | Decidir si una contraseña es correcta. |

**Argumento:** separar así permite **probar la lógica sin abrir ventanas** (tests de `AuthService` con
JUnit), cambiar la base de datos (MySQL, SQLite, memoria) sin tocar la interfaz, y que cada clase sea
corta y fácil de explicar en clase. Es el patrón *MVC + Service + Repository* que se espera en un
proyecto académico.

**Estructura de archivos sugerida**

```
src/main/java/com/example/auth/
├── App.java
├── controller/
│   ├── SignInController.java
│   ├── SignUpController.java
│   └── ChangePasswordController.java
├── model/
│   ├── User.java
│   └── SignUpRequest.java
├── service/
│   ├── AuthService.java
│   ├── AuthException.java
│   └── ValidationException.java
├── repository/
│   ├── UserRepository.java
│   ├── JdbcUserRepository.java
│   ├── InMemoryUserRepository.java
│   └── Database.java
└── util/
    ├── Validator.java
    ├── PasswordHasher.java
    ├── Session.java
    ├── SceneNavigator.java
    └── Views.java
src/main/resources/com/example/auth/
├── sign_in.fxml
├── sign_up.fxml
├── change_password.fxml
├── home.fxml          ← ventana principal tras iniciar sesión (la creas tú)
└── auth.css
```

---

## 2. Modelo

### `User`

Representa un usuario **tal como está guardado** en la base de datos.

```java
package com.example.auth.model;

/**
 * Usuario registrado.
 *
 * Nunca guarda la contraseña en texto plano: solo su hash (ver PasswordHasher).
 * Es un record (Java 16+): inmutable, con constructor, getters, equals y toString generados.
 */
public record User(
        long id,              // Clave primaria generada por la BD
        String firstName,
        String lastName,
        String email,         // Siempre en minúsculas (se normaliza en AuthService)
        String passwordHash,  // Formato "iteraciones:salt:hash" (Base64)
        String state,
        String city,
        String street,
        String zipCode
) {
    /** Nombre completo, útil para mostrar "Hola, Ana López" en la ventana principal. */
    public String fullName() {
        return firstName + " " + lastName;
    }
}
```

**Argumento:** usar `record` evita escribir decenas de líneas de getters y garantiza inmutabilidad;
un usuario no debería cambiar "por accidente" en memoria. Si tu versión de Java es anterior a la 16,
conviértelo en una clase normal con campos `private final` y getters.

### `SignUpRequest`

Agrupa los datos que llegan del formulario de registro **antes** de validarlos.

```java
package com.example.auth.model;

/**
 * Datos crudos del formulario de registro.
 *
 * Se separa de User porque todavía no tiene id ni hash, y la contraseña
 * viene en texto plano (solo vive en memoria el tiempo justo para validarla y hashearla).
 */
public record SignUpRequest(
        String firstName,
        String lastName,
        String email,
        String password,
        String confirmPassword,
        String state,
        String city,
        String street,
        String zipCode,
        boolean acceptedTerms
) {}
```

**Argumento:** pasar un único objeto en vez de 10 parámetros sueltos a `signUp(...)` evita errores
de orden (confundir `city` con `state`) y permite añadir campos sin cambiar la firma del método.

---

## 3. Excepciones

### `AuthException`

```java
package com.example.auth.service;

/**
 * Error "esperado" de autenticación: credenciales incorrectas, email ya registrado, etc.
 *
 * Es checked (extends Exception) para que el compilador obligue al controlador
 * a gestionarla y mostrar el mensaje al usuario.
 * El mensaje está pensado para mostrarse directamente en pantalla.
 */
public class AuthException extends Exception {
    public AuthException(String message) {
        super(message);
    }
}
```

### `ValidationException`

```java
package com.example.auth.service;

/**
 * Error de validación asociado a un campo concreto del formulario.
 *
 * El nombre del campo permite al controlador resaltar en rojo el input correcto.
 */
public class ValidationException extends AuthException {

    /** Campos que pueden fallar. El controlador los traduce a su TextField. */
    public enum Field {
        FIRST_NAME, LAST_NAME, EMAIL, PASSWORD, CONFIRM_PASSWORD,
        CURRENT_PASSWORD, STATE, CITY, STREET, ZIP_CODE, TERMS
    }

    private final Field field;

    public ValidationException(Field field, String message) {
        super(message);
        this.field = field;
    }

    public Field getField() {
        return field;
    }
}
```

**Argumento:** distinguir entre *error de validación* (culpa del formulario, se señala un campo) y
*error de autenticación* (credenciales malas, no se señala ninguno) permite mostrar una interfaz más
clara. Los errores **inesperados** (BD caída, bug) siguen siendo `RuntimeException` y se tratan aparte.

---

## 4. Utilidades

### `Validator`

Funciones puras (sin estado) para comprobar formatos.

```java
package com.example.auth.util;

import java.util.regex.Pattern;

/** Reglas de formato reutilizables. Todos los métodos son estáticos y sin efectos secundarios. */
public final class Validator {

    // Suficiente para un proyecto: algo@algo.dominio, sin espacios.
    private static final Pattern EMAIL = Pattern.compile("^[^\\s@]+@[^\\s@]+\\.[^\\s@]{2,}$");

    // Código postal de EE. UU.: 12345 o 12345-6789 (los campos State/Zip del formulario siguen ese formato).
    private static final Pattern ZIP = Pattern.compile("^\\d{5}(-\\d{4})?$");

    public static final int MIN_PASSWORD_LENGTH = 8;

    private Validator() {} // Clase de utilidades: no se instancia

    /** true si el texto es null, vacío o solo espacios. */
    public static boolean isBlank(String s) {
        return s == null || s.isBlank();
    }

    /** true si el email tiene un formato válido. */
    public static boolean isValidEmail(String email) {
        return email != null && EMAIL.matcher(email.trim()).matches();
    }

    /**
     * Contraseña "fuerte" para el proyecto: mínimo 8 caracteres, al menos una letra y un número.
     * Coincide con el texto de ayuda "At least 8 characters" de change_password.fxml.
     */
    public static boolean isStrongPassword(String password) {
        if (password == null || password.length() < MIN_PASSWORD_LENGTH) return false;
        boolean hasLetter = password.chars().anyMatch(Character::isLetter);
        boolean hasDigit = password.chars().anyMatch(Character::isDigit);
        return hasLetter && hasDigit;
    }

    /** true si el código postal tiene formato 12345 o 12345-6789. */
    public static boolean isValidZip(String zip) {
        return zip != null && ZIP.matcher(zip.trim()).matches();
    }

    /** Normaliza un email: sin espacios y en minúsculas, para que "Ana@X.com" y "ana@x.com" sean el mismo. */
    public static String normalizeEmail(String email) {
        return email == null ? "" : email.trim().toLowerCase();
    }
}
```

**Argumento:** centralizar las reglas evita que el registro acepte una contraseña de 6 caracteres y el
cambio de contraseña exija 8. Al ser métodos puros son triviales de testear.

### `PasswordHasher`

Convierte contraseñas en hashes seguros y los verifica. **Sin librerías externas** (solo el JDK).

```java
package com.example.auth.util;

import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;
import java.security.GeneralSecurityException;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

/**
 * Hash de contraseñas con PBKDF2-HMAC-SHA256.
 *
 * Formato guardado en BD: "iteraciones:saltBase64:hashBase64"
 * Guardar las iteraciones permite subirlas en el futuro sin invalidar los hashes antiguos.
 */
public final class PasswordHasher {

    private static final String ALGORITHM = "PBKDF2WithHmacSHA256";
    private static final int ITERATIONS = 210_000; // Recomendación OWASP para PBKDF2-SHA256
    private static final int SALT_BYTES = 16;
    private static final int KEY_BITS = 256;
    private static final SecureRandom RANDOM = new SecureRandom();

    private PasswordHasher() {}

    /**
     * Genera el hash de una contraseña nueva.
     *
     * @param password contraseña en texto plano
     * @return cadena lista para guardar en la columna password_hash
     */
    public static String hash(String password) {
        byte[] salt = new byte[SALT_BYTES];
        RANDOM.nextBytes(salt); // Salt aleatorio: dos usuarios con la misma contraseña tienen hashes distintos
        byte[] hash = pbkdf2(password, salt, ITERATIONS);
        Base64.Encoder b64 = Base64.getEncoder();
        return ITERATIONS + ":" + b64.encodeToString(salt) + ":" + b64.encodeToString(hash);
    }

    /**
     * Comprueba si una contraseña coincide con un hash guardado.
     *
     * @param password contraseña escrita por el usuario
     * @param stored   valor de la columna password_hash
     * @return true si coincide
     */
    public static boolean verify(String password, String stored) {
        String[] parts = stored.split(":");
        if (parts.length != 3) return false; // Hash corrupto o de otro formato

        int iterations = Integer.parseInt(parts[0]);
        byte[] salt = Base64.getDecoder().decode(parts[1]);
        byte[] expected = Base64.getDecoder().decode(parts[2]);
        byte[] actual = pbkdf2(password, salt, iterations);

        // Comparación en tiempo constante: evita deducir el hash midiendo cuánto tarda la comparación
        return MessageDigest.isEqual(expected, actual);
    }

    private static byte[] pbkdf2(String password, byte[] salt, int iterations) {
        PBEKeySpec spec = new PBEKeySpec(password.toCharArray(), salt, iterations, KEY_BITS);
        try {
            return SecretKeyFactory.getInstance(ALGORITHM).generateSecret(spec).getEncoded();
        } catch (GeneralSecurityException e) {
            // Solo ocurre si la JVM no soporta el algoritmo (todas las JVM modernas lo soportan)
            throw new IllegalStateException("PBKDF2 no disponible", e);
        } finally {
            spec.clearPassword(); // Borra la copia interna de la contraseña
        }
    }
}
```

**Argumento:**
- **Nunca** se guarda la contraseña en texto plano ni con MD5/SHA-1 simples: si alguien roba la BD,
  podría leerla o romperla en segundos.
- PBKDF2 es **lento a propósito** (210 000 iteraciones) para que un atacante no pueda probar millones
  de contraseñas por segundo.
- El *salt* aleatorio impide usar tablas precalculadas (*rainbow tables*).
- Está incluido en el JDK, así que no añade dependencias al proyecto. Si tu profesor permite librerías,
  **BCrypt** (`org.mindrot:jbcrypt`) es una alternativa igual de válida.

### `Session`

Guarda quién ha iniciado sesión mientras la aplicación está abierta.

```java
package com.example.auth.util;

import com.example.auth.model.User;

/**
 * Sesión en memoria de la aplicación de escritorio.
 *
 * En una app de escritorio hay un solo usuario a la vez, así que basta un campo estático.
 * (En un servidor web esto sería un token o una cookie, no un static.)
 */
public final class Session {

    private static User currentUser;

    private Session() {}

    /** Inicia sesión con el usuario autenticado. */
    public static void start(User user) {
        currentUser = user;
    }

    /** Cierra la sesión. */
    public static void end() {
        currentUser = null;
    }

    /** Usuario actual o null si nadie ha iniciado sesión. */
    public static User getCurrentUser() {
        return currentUser;
    }

    /** Atajo para comprobar si hay sesión activa. */
    public static boolean isActive() {
        return currentUser != null;
    }

    /** Actualiza el usuario guardado (por ejemplo, tras cambiar la contraseña). */
    public static void refresh(User user) {
        currentUser = user;
    }
}
```

**Argumento:** `ChangePasswordController` necesita saber **de quién** es la contraseña que cambia
sin volver a pedir el email. `Session` resuelve eso de forma simple.

### `Views` y `SceneNavigator`

```java
package com.example.auth.util;

/** Rutas de los FXML en un único sitio: si mueves un archivo, solo cambias esta clase. */
public final class Views {
    public static final String SIGN_IN = "/com/example/auth/sign_in.fxml";
    public static final String SIGN_UP = "/com/example/auth/sign_up.fxml";
    public static final String CHANGE_PASSWORD = "/com/example/auth/change_password.fxml";
    public static final String HOME = "/com/example/auth/home.fxml";

    private Views() {}
}
```

```java
package com.example.auth.util;

import javafx.fxml.FXMLLoader;
import javafx.scene.Node;
import javafx.scene.Parent;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.net.URL;

/** Cambia la ventana actual por otra vista FXML. */
public final class SceneNavigator {

    private SceneNavigator() {}

    /**
     * Sustituye el contenido de la ventana que contiene a {@code source} por la vista indicada.
     *
     * @param source cualquier nodo de la vista actual (por ejemplo, el botón pulsado)
     * @param fxml   ruta del FXML, normalmente una constante de {@link Views}
     */
    public static void navigate(Node source, String fxml) {
        URL url = SceneNavigator.class.getResource(fxml);
        if (url == null) {
            throw new IllegalArgumentException("No se encontró el FXML: " + fxml);
        }
        try {
            Parent root = FXMLLoader.load(url);
            // setRoot reutiliza la misma Scene y Stage: la ventana conserva tamaño y posición
            source.getScene().setRoot(root);
        } catch (IOException e) {
            throw new UncheckedIOException("Error al cargar " + fxml, e);
        }
    }
}
```

**Argumento:** cambiar solo el *root* de la escena (en vez de abrir un `Stage` nuevo) evita ventanas
duplicadas y mantiene el tamaño que el usuario eligió. Como cada FXML declara
`stylesheets="@auth.css"` en su raíz, el estilo viaja con la vista.

---

## 5. Capa de datos: `UserRepository`

### Interfaz

```java
package com.example.auth.repository;

import com.example.auth.model.User;
import java.util.Optional;

/**
 * Acceso a los usuarios guardados.
 *
 * Es una interfaz para poder tener dos implementaciones:
 *  - JdbcUserRepository: base de datos real.
 *  - InMemoryUserRepository: para pruebas o para entregar sin BD.
 */
public interface UserRepository {

    /**
     * Busca un usuario por email.
     * @param email ya normalizado (minúsculas, sin espacios)
     * @return el usuario, o Optional.empty() si no existe
     */
    Optional<User> findByEmail(String email);

    /** Busca un usuario por su id. */
    Optional<User> findById(long id);

    /** true si ya hay una cuenta con ese email. */
    boolean existsByEmail(String email);

    /**
     * Inserta un usuario nuevo.
     * @param user usuario sin id (el id que traiga se ignora)
     * @return el mismo usuario con el id generado por la BD
     */
    User save(User user);

    /**
     * Reemplaza el hash de contraseña de un usuario.
     * @return true si se actualizó una fila
     */
    boolean updatePassword(long userId, String newPasswordHash);
}
```

**Argumento:** `Optional` obliga a quien llama a contemplar el caso "no existe", evitando
`NullPointerException`. Programar contra la interfaz permite arrancar el proyecto con la versión en
memoria y conectar la base de datos después sin tocar `AuthService` ni los controladores.

### Tabla SQL

```sql
CREATE TABLE users (
    id            INTEGER PRIMARY KEY AUTO_INCREMENT,   -- En SQLite: INTEGER PRIMARY KEY AUTOINCREMENT
    first_name    VARCHAR(80)  NOT NULL,
    last_name     VARCHAR(80)  NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,          -- UNIQUE: la BD también impide duplicados
    password_hash VARCHAR(255) NOT NULL,
    state         VARCHAR(80)  NOT NULL,
    city          VARCHAR(80)  NOT NULL,
    street        VARCHAR(255) NOT NULL,
    zip_code      VARCHAR(10)  NOT NULL,
    created_at    TIMESTAMP    DEFAULT CURRENT_TIMESTAMP
);
```

### `Database`

```java
package com.example.auth.repository;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

/** Punto único para obtener conexiones. Cambia URL/usuario/clave según tu BD. */
public final class Database {

    private static final String URL = "jdbc:mysql://localhost:3306/auth_app"; // o "jdbc:sqlite:auth.db"
    private static final String USER = "root";
    private static final String PASSWORD = "";

    private Database() {}

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
}
```

> Recuerda añadir el driver JDBC (`mysql-connector-j` o `sqlite-jdbc`) a tu `pom.xml`/`build.gradle`.

### `JdbcUserRepository`

```java
package com.example.auth.repository;

import com.example.auth.model.User;

import java.sql.*;
import java.util.Optional;

/** Implementación con JDBC puro. Todas las consultas usan PreparedStatement. */
public class JdbcUserRepository implements UserRepository {

    @Override
    public Optional<User> findByEmail(String email) {
        String sql = "SELECT * FROM users WHERE email = ?";
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(sql)) {
            ps.setString(1, email);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(map(rs)) : Optional.empty();
            }
        } catch (SQLException e) {
            throw new IllegalStateException("Error consultando usuario por email", e);
        }
    }

    @Override
    public Optional<User> findById(long id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(sql)) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(map(rs)) : Optional.empty();
            }
        } catch (SQLException e) {
            throw new IllegalStateException("Error consultando usuario por id", e);
        }
    }

    @Override
    public boolean existsByEmail(String email) {
        return findByEmail(email).isPresent();
    }

    @Override
    public User save(User u) {
        String sql = """
                INSERT INTO users (first_name, last_name, email, password_hash, state, city, street, zip_code)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                """;
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, u.firstName());
            ps.setString(2, u.lastName());
            ps.setString(3, u.email());
            ps.setString(4, u.passwordHash());
            ps.setString(5, u.state());
            ps.setString(6, u.city());
            ps.setString(7, u.street());
            ps.setString(8, u.zipCode());
            ps.executeUpdate();

            try (ResultSet keys = ps.getGeneratedKeys()) {
                keys.next();
                long id = keys.getLong(1);
                return new User(id, u.firstName(), u.lastName(), u.email(), u.passwordHash(),
                        u.state(), u.city(), u.street(), u.zipCode());
            }
        } catch (SQLException e) {
            throw new IllegalStateException("Error guardando usuario", e);
        }
    }

    @Override
    public boolean updatePassword(long userId, String newPasswordHash) {
        String sql = "UPDATE users SET password_hash = ? WHERE id = ?";
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(sql)) {
            ps.setString(1, newPasswordHash);
            ps.setLong(2, userId);
            return ps.executeUpdate() == 1;
        } catch (SQLException e) {
            throw new IllegalStateException("Error actualizando contraseña", e);
        }
    }

    /** Convierte la fila actual del ResultSet en un User. */
    private User map(ResultSet rs) throws SQLException {
        return new User(
                rs.getLong("id"),
                rs.getString("first_name"),
                rs.getString("last_name"),
                rs.getString("email"),
                rs.getString("password_hash"),
                rs.getString("state"),
                rs.getString("city"),
                rs.getString("street"),
                rs.getString("zip_code"));
    }
}
```

**Argumento:**
- `PreparedStatement` con `?` impide la **inyección SQL** (escribir `' OR 1=1 --` en el email no hace nada).
- `try-with-resources` cierra conexiones y cursores aunque haya errores, evitando agotar conexiones.
- La `SQLException` se envuelve en una excepción no comprobada: para el usuario un fallo de BD no es
  algo que pueda corregir en el formulario, así que se trata como error inesperado.

### `InMemoryUserRepository` (opcional, para empezar sin BD)

```java
package com.example.auth.repository;

import com.example.auth.model.User;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/** Guarda usuarios en memoria. Se pierden al cerrar la app: ideal para pruebas y demos. */
public class InMemoryUserRepository implements UserRepository {

    private final Map<Long, User> users = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong(1);

    @Override
    public Optional<User> findByEmail(String email) {
        return users.values().stream().filter(u -> u.email().equals(email)).findFirst();
    }

    @Override
    public Optional<User> findById(long id) {
        return Optional.ofNullable(users.get(id));
    }

    @Override
    public boolean existsByEmail(String email) {
        return findByEmail(email).isPresent();
    }

    @Override
    public User save(User u) {
        long id = nextId.getAndIncrement();
        User saved = new User(id, u.firstName(), u.lastName(), u.email(), u.passwordHash(),
                u.state(), u.city(), u.street(), u.zipCode());
        users.put(id, saved);
        return saved;
    }

    @Override
    public boolean updatePassword(long userId, String newPasswordHash) {
        User u = users.get(userId);
        if (u == null) return false;
        users.put(userId, new User(u.id(), u.firstName(), u.lastName(), u.email(), newPasswordHash,
                u.state(), u.city(), u.street(), u.zipCode()));
        return true;
    }
}
```

Se usan colecciones *concurrentes* porque el servicio se llama desde un hilo en segundo plano
(ver sección 7).

---

## 6. Capa de negocio: `AuthService`

Es el **corazón** del backend: aquí viven todas las reglas.

```java
package com.example.auth.service;

import com.example.auth.model.SignUpRequest;
import com.example.auth.model.User;
import com.example.auth.repository.UserRepository;
import com.example.auth.service.ValidationException.Field;
import com.example.auth.util.PasswordHasher;
import com.example.auth.util.Session;
import com.example.auth.util.Validator;

/**
 * Operaciones de autenticación: iniciar sesión, registrarse, cambiar contraseña y cerrar sesión.
 *
 * No depende de JavaFX: se puede probar con JUnit sin abrir ninguna ventana.
 */
public class AuthService {

    private final UserRepository users;

    /** El repositorio se inyecta por constructor: así en los tests se pasa InMemoryUserRepository. */
    public AuthService(UserRepository users) {
        this.users = users;
    }

    // ------------------------------------------------------------------ signIn

    /**
     * Inicia sesión.
     *
     * @param email    email escrito por el usuario (se normaliza)
     * @param password contraseña en texto plano
     * @return el usuario autenticado (también queda guardado en Session)
     * @throws ValidationException si algún campo está vacío o el email no tiene formato válido
     * @throws AuthException       si el email no existe o la contraseña no coincide
     */
    public User signIn(String email, String password) throws AuthException {
        if (Validator.isBlank(email)) {
            throw new ValidationException(Field.EMAIL, "Introduce tu email.");
        }
        if (!Validator.isValidEmail(email)) {
            throw new ValidationException(Field.EMAIL, "El email no tiene un formato válido.");
        }
        if (Validator.isBlank(password)) {
            throw new ValidationException(Field.PASSWORD, "Introduce tu contraseña.");
        }

        User user = users.findByEmail(Validator.normalizeEmail(email))
                .filter(u -> PasswordHasher.verify(password, u.passwordHash()))
                // Mismo mensaje si el email no existe o la contraseña falla (ver argumento abajo)
                .orElseThrow(() -> new AuthException("Email o contraseña incorrectos."));

        Session.start(user);
        return user;
    }

    // ------------------------------------------------------------------ signUp

    /**
     * Registra una cuenta nueva.
     *
     * @param req datos del formulario
     * @return el usuario creado (con id). No inicia sesión automáticamente.
     * @throws ValidationException si algún campo es inválido
     * @throws AuthException       si el email ya está registrado
     */
    public User signUp(SignUpRequest req) throws AuthException {
        validateSignUp(req);

        String email = Validator.normalizeEmail(req.email());
        if (users.existsByEmail(email)) {
            throw new ValidationException(Field.EMAIL, "Ya existe una cuenta con ese email.");
        }

        User toSave = new User(
                0, // La BD asigna el id real
                req.firstName().trim(),
                req.lastName().trim(),
                email,
                PasswordHasher.hash(req.password()),
                req.state().trim(),
                req.city().trim(),
                req.street().trim(),
                req.zipCode().trim());

        return users.save(toSave);
    }

    /**
     * Comprueba todos los campos del registro en el mismo orden en que aparecen en pantalla,
     * para que el primer error mostrado sea el del campo más arriba.
     */
    private void validateSignUp(SignUpRequest r) throws ValidationException {
        requireNotBlank(r.firstName(), Field.FIRST_NAME, "Introduce tu nombre.");
        requireNotBlank(r.lastName(), Field.LAST_NAME, "Introduce tus apellidos.");

        requireNotBlank(r.email(), Field.EMAIL, "Introduce tu email.");
        if (!Validator.isValidEmail(r.email())) {
            throw new ValidationException(Field.EMAIL, "El email no tiene un formato válido.");
        }

        validateNewPassword(r.password(), r.confirmPassword(), Field.PASSWORD);

        requireNotBlank(r.state(), Field.STATE, "Introduce tu estado.");
        requireNotBlank(r.city(), Field.CITY, "Introduce tu ciudad.");
        requireNotBlank(r.street(), Field.STREET, "Introduce tu dirección.");
        if (!Validator.isValidZip(r.zipCode())) {
            throw new ValidationException(Field.ZIP_CODE, "El código postal debe tener 5 dígitos.");
        }

        if (!r.acceptedTerms()) {
            throw new ValidationException(Field.TERMS, "Debes aceptar los términos y la política de privacidad.");
        }
    }

    // ------------------------------------------------------------------ changePassword

    /**
     * Cambia la contraseña del usuario con sesión iniciada.
     *
     * @param currentPassword contraseña actual (se exige para confirmar que es el dueño de la cuenta)
     * @param newPassword     contraseña nueva
     * @param confirmPassword repetición de la nueva
     * @throws AuthException       si no hay sesión
     * @throws ValidationException si la actual es incorrecta o la nueva no cumple las reglas
     */
    public void changePassword(String currentPassword, String newPassword, String confirmPassword)
            throws AuthException {

        User user = Session.getCurrentUser();
        if (user == null) {
            throw new AuthException("Tu sesión ha expirado. Vuelve a iniciar sesión.");
        }

        requireNotBlank(currentPassword, Field.CURRENT_PASSWORD, "Introduce tu contraseña actual.");
        if (!PasswordHasher.verify(currentPassword, user.passwordHash())) {
            throw new ValidationException(Field.CURRENT_PASSWORD, "La contraseña actual no es correcta.");
        }

        validateNewPassword(newPassword, confirmPassword, Field.PASSWORD);
        if (newPassword.equals(currentPassword)) {
            throw new ValidationException(Field.PASSWORD, "La nueva contraseña debe ser distinta de la actual.");
        }

        String newHash = PasswordHasher.hash(newPassword);
        if (!users.updatePassword(user.id(), newHash)) {
            throw new AuthException("No se pudo actualizar la contraseña. Inténtalo de nuevo.");
        }

        // Mantiene la sesión coherente con la BD (si no, el siguiente cambio compararía con el hash viejo)
        users.findById(user.id()).ifPresent(Session::refresh);
    }

    // ------------------------------------------------------------------ signOut

    /** Cierra la sesión actual. */
    public void signOut() {
        Session.end();
    }

    // ------------------------------------------------------------------ helpers

    /** Reglas comunes de contraseña nueva (registro y cambio de contraseña). */
    private void validateNewPassword(String password, String confirm, Field field)
            throws ValidationException {
        requireNotBlank(password, field, "Introduce una contraseña.");
        if (!Validator.isStrongPassword(password)) {
            throw new ValidationException(field,
                    "La contraseña debe tener al menos " + Validator.MIN_PASSWORD_LENGTH
                            + " caracteres e incluir letras y números.");
        }
        if (!password.equals(confirm)) {
            throw new ValidationException(Field.CONFIRM_PASSWORD, "Las contraseñas no coinciden.");
        }
    }

    private void requireNotBlank(String value, Field field, String message) throws ValidationException {
        if (Validator.isBlank(value)) {
            throw new ValidationException(field, message);
        }
    }
}
```

**Argumentos de diseño**

- **Mensaje genérico en `signIn`.** Decir "ese email no existe" permitiría a cualquiera averiguar
  qué emails están registrados (*enumeración de usuarios*). Con "Email o contraseña incorrectos"
  no se revela nada. *(El registro sí dice "ya existe una cuenta", porque es inevitable: el usuario
  necesita saberlo. En aplicaciones reales se mitiga enviando un email en vez de mostrarlo.)*
- **`signUp` no inicia sesión.** Tras registrarse, la app vuelve a *Sign In*: así se comprueba que el
  usuario recuerda la contraseña y el flujo es más fácil de demostrar. Si prefieres entrar
  directamente, añade `Session.start(saved)` antes del `return`.
- **`changePassword` pide la contraseña actual.** Si alguien deja el ordenador desbloqueado, otra
  persona no puede cambiarle la contraseña.
- **Validación en el servicio, no en el controlador.** Es la única fuente de verdad: si mañana añades
  otra pantalla (o una API), las reglas son las mismas.
- **Orden de validación = orden visual.** El usuario corrige de arriba abajo sin sorpresas.

### ¿Y "Forgot Password?"?

Un restablecimiento real necesita enviar un email con un enlace o código, lo que implica servidor de
correo (JavaMail/Jakarta Mail) y una tabla de *tokens* con caducidad. Para un proyecto de clase hay dos
opciones razonables:

1. **Mostrar un aviso** ("Contacta con el administrador para restablecer tu contraseña") — recomendado
   si no está en los requisitos.
2. Implementarlo con dos métodos más en el servicio:
   - `requestPasswordReset(String email)`: genera un código aleatorio de 6 dígitos, guarda su hash y
     la caducidad (15 min) y lo envía por email. **Siempre** responde "Si el email existe, te hemos
     enviado un código" (por el mismo motivo de enumeración).
   - `resetPassword(String email, String code, String newPassword, String confirm)`: verifica código
     y caducidad, aplica `validateNewPassword` y llama a `updatePassword`.

El controlador de la sección 7 usa la opción 1.

---

## 7. Controladores JavaFX

### Conceptos comunes

- **`@FXML`** conecta un campo o método con el FXML. El nombre del campo debe coincidir con el
  `fx:id`, y el del método con el `onAction="#metodo"`.
- **`initialize()`** lo llama JavaFX automáticamente después de inyectar los `@FXML`. Es el lugar
  para configurar *listeners* y *bindings*.
- **Hilo de la interfaz.** Hashear con PBKDF2 tarda ~0,2 s y consultar la BD puede tardar más. Si se
  hace en el hilo de JavaFX la ventana se congela. Por eso las llamadas al servicio se envuelven en un
  `Task` que corre en segundo plano; `setOnSucceeded`/`setOnFailed` vuelven al hilo de JavaFX
  automáticamente, así que ahí sí se puede tocar la interfaz.
- **Una única instancia de `AuthService`.** Se crea en `App` y los controladores la obtienen con
  `App.authService()`. Es lo más simple; en proyectos grandes se usaría inyección de dependencias.

```java
package com.example.auth;

import com.example.auth.repository.JdbcUserRepository;
import com.example.auth.service.AuthService;
import com.example.auth.util.Views;
import javafx.application.Application;
import javafx.fxml.FXMLLoader;
import javafx.scene.Parent;
import javafx.scene.Scene;
import javafx.stage.Stage;

public class App extends Application {

    // Cambia a new InMemoryUserRepository() para probar sin base de datos
    private static final AuthService AUTH_SERVICE = new AuthService(new JdbcUserRepository());

    public static AuthService authService() {
        return AUTH_SERVICE;
    }

    @Override
    public void start(Stage stage) throws Exception {
        Parent root = FXMLLoader.load(App.class.getResource(Views.SIGN_IN));
        stage.setTitle("Mi aplicación");
        stage.setScene(new Scene(root));
        stage.setMinWidth(480);
        stage.setMinHeight(600);
        stage.show();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
```

### Helper compartido: `FormErrors`

Los tres controladores muestran errores igual, así que la lógica se comparte.

```java
package com.example.auth.controller;

import javafx.scene.Node;
import javafx.scene.control.Label;

/** Muestra y oculta errores en los formularios de autenticación. */
final class FormErrors {

    private static final String ERROR_CLASS = "error";

    private FormErrors() {}

    /** Muestra el mensaje en el label (y hace que ocupe espacio en el layout). */
    static void show(Label errorLabel, String message) {
        errorLabel.setText(message);
        errorLabel.setManaged(true);
        errorLabel.setVisible(true);
    }

    /** Oculta el label y quita el borde rojo de todos los campos indicados. */
    static void clear(Label errorLabel, Node... fields) {
        errorLabel.setManaged(false);
        errorLabel.setVisible(false);
        for (Node field : fields) {
            container(field).getStyleClass().remove(ERROR_CLASS);
        }
    }

    /**
     * Resalta un campo en rojo y le da el foco.
     * Se marca el HBox padre (.auth-input) porque es quien dibuja el borde redondeado.
     */
    static void highlight(Node field) {
        Node box = container(field);
        if (!box.getStyleClass().contains(ERROR_CLASS)) {
            box.getStyleClass().add(ERROR_CLASS);
        }
        field.requestFocus();
    }

    /** Si el campo está dentro de un .auth-input devuelve ese HBox; si no, el propio nodo. */
    private static Node container(Node field) {
        Node parent = field.getParent();
        return parent != null && parent.getStyleClass().contains("auth-input") ? parent : field;
    }
}
```

**Argumento:** `managed=false` + `visible=false` hace que el label oculto **no ocupe hueco**; al
mostrar un error la tarjeta crece justo lo necesario.

---

### 7.1 `SignInController`

**`fx:id` usados:** `emailField`, `passwordField`, `forgotPasswordLink`, `signInButton`, `signUpLink`,
`errorLabel` (nuevo, ver sección 8).

```java
package com.example.auth.controller;

import com.example.auth.App;
import com.example.auth.model.User;
import com.example.auth.service.AuthException;
import com.example.auth.service.AuthService;
import com.example.auth.service.ValidationException;
import com.example.auth.util.SceneNavigator;
import com.example.auth.util.Views;
import javafx.concurrent.Task;
import javafx.fxml.FXML;
import javafx.scene.control.*;

public class SignInController {

    @FXML private TextField emailField;
    @FXML private PasswordField passwordField;
    @FXML private Hyperlink forgotPasswordLink;
    @FXML private Button signInButton;
    @FXML private Hyperlink signUpLink;
    @FXML private Label errorLabel;

    private final AuthService authService = App.authService();

    /**
     * Se ejecuta al cargar la vista.
     * - Borra el error en cuanto el usuario vuelve a escribir (feedback inmediato).
     * - Pone el foco en el email para poder escribir sin hacer clic.
     */
    @FXML
    private void initialize() {
        emailField.textProperty().addListener((obs, old, val) -> clearError());
        passwordField.textProperty().addListener((obs, old, val) -> clearError());
        // runLater: el foco solo se puede pedir cuando la vista ya está en la escena
        javafx.application.Platform.runLater(emailField::requestFocus);
    }

    /**
     * Botón "Sign In" (o Enter, porque es defaultButton).
     * Llama a AuthService.signIn en segundo plano y, si va bien, navega a la ventana principal.
     */
    @FXML
    private void handleSignIn() {
        clearError();
        String email = emailField.getText();
        String password = passwordField.getText();

        Task<User> task = new Task<>() {
            @Override
            protected User call() throws Exception {
                return authService.signIn(email, password);
            }
        };

        // Desactiva el botón mientras trabaja: evita dobles clics y peticiones duplicadas
        signInButton.disableProperty().bind(task.runningProperty());

        task.setOnSucceeded(e -> SceneNavigator.navigate(signInButton, Views.HOME));
        task.setOnFailed(e -> {
            passwordField.clear(); // Por seguridad, no se deja la contraseña escrita tras un fallo
            handleError(task.getException());
        });

        startInBackground(task);
    }

    /** Enlace "Forgot Password?". Ver sección 6: versión simple con aviso. */
    @FXML
    private void handleForgotPassword() {
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Recuperar contraseña");
        alert.setHeaderText(null);
        alert.setContentText("Contacta con el administrador para restablecer tu contraseña.");
        alert.initOwner(signInButton.getScene().getWindow());
        alert.showAndWait();
    }

    /** Enlace "Sign Up": abre el registro. */
    @FXML
    private void goToSignUp() {
        SceneNavigator.navigate(signUpLink, Views.SIGN_UP);
    }

    // ---------------------------------------------------------------- helpers

    /** Traduce una excepción en un mensaje visible y, si procede, resalta el campo culpable. */
    private void handleError(Throwable error) {
        if (error instanceof ValidationException ve) {
            FormErrors.show(errorLabel, ve.getMessage());
            switch (ve.getField()) {
                case EMAIL -> FormErrors.highlight(emailField);
                case PASSWORD -> FormErrors.highlight(passwordField);
                default -> { }
            }
        } else if (error instanceof AuthException ae) {
            FormErrors.show(errorLabel, ae.getMessage());
            passwordField.requestFocus();
        } else {
            // Error inesperado (BD caída, bug): mensaje genérico al usuario y detalle en consola
            error.printStackTrace();
            FormErrors.show(errorLabel, "Ha ocurrido un error inesperado. Inténtalo más tarde.");
        }
    }

    private void clearError() {
        FormErrors.clear(errorLabel, emailField, passwordField);
    }

    /** Lanza la tarea en un hilo daemon (no impide cerrar la app si sigue ejecutándose). */
    static void startInBackground(Task<?> task) {
        Thread thread = new Thread(task, "auth-task");
        thread.setDaemon(true);
        thread.start();
    }
}
```

| Método | Disparado por | Qué hace | Por qué así |
|---|---|---|---|
| `initialize()` | JavaFX al cargar | Listeners que borran el error; foco en email | Feedback inmediato y teclado listo |
| `handleSignIn()` | Botón *Sign In* / Enter | Autentica en segundo plano y navega a Home | No congela la ventana; evita doble envío |
| `handleForgotPassword()` | Enlace *Forgot Password?* | Muestra un aviso | Restablecer de verdad requiere email (sección 6) |
| `goToSignUp()` | Enlace *Sign Up* | Cambia a la vista de registro | Navegación sin abrir ventanas nuevas |
| `handleError(Throwable)` | Fallo del `Task` | Muestra mensaje y resalta campo | Distingue validación, credenciales e imprevistos |
| `clearError()` | Escribir / reintentar | Oculta mensaje y bordes rojos | Interfaz limpia en cada intento |
| `startInBackground(Task)` | Interno | Arranca un hilo daemon | Reutilizado por los otros controladores |

---

### 7.2 `SignUpController`

**`fx:id` usados:** `firstNameField`, `lastNameField`, `emailField`, `passwordField`,
`confirmPasswordField`, `stateField`, `cityField`, `streetField`, `zipField`, `termsCheckBox`,
`signUpButton`, `signInLink`, `errorLabel`.

> Tu versión del formulario tiene *First Name* y *Last Name* separados y los campos de dirección;
> ponles estos `fx:id` (o cambia los nombres aquí).

```java
package com.example.auth.controller;

import com.example.auth.App;
import com.example.auth.model.SignUpRequest;
import com.example.auth.model.User;
import com.example.auth.service.AuthException;
import com.example.auth.service.AuthService;
import com.example.auth.service.ValidationException;
import com.example.auth.util.SceneNavigator;
import com.example.auth.util.Views;
import javafx.concurrent.Task;
import javafx.fxml.FXML;
import javafx.scene.Node;
import javafx.scene.control.*;

public class SignUpController {

    @FXML private TextField firstNameField;
    @FXML private TextField lastNameField;
    @FXML private TextField emailField;
    @FXML private PasswordField passwordField;
    @FXML private PasswordField confirmPasswordField;
    @FXML private TextField stateField;
    @FXML private TextField cityField;
    @FXML private TextField streetField;
    @FXML private TextField zipField;
    @FXML private CheckBox termsCheckBox;
    @FXML private Button signUpButton;
    @FXML private Hyperlink signInLink;
    @FXML private Label errorLabel;

    private final AuthService authService = App.authService();

    /** Todos los campos de texto, para limpiar errores en bloque. */
    private Node[] allFields() {
        return new Node[]{firstNameField, lastNameField, emailField, passwordField,
                confirmPasswordField, stateField, cityField, streetField, zipField};
    }

    /**
     * - Cualquier cambio en un campo borra el error mostrado.
     * - El código postal solo admite dígitos y guion, con máximo 10 caracteres.
     */
    @FXML
    private void initialize() {
        for (Node field : allFields()) {
            ((TextInputControl) field).textProperty().addListener((obs, old, val) -> clearError());
        }
        termsCheckBox.selectedProperty().addListener((obs, old, val) -> clearError());

        // TextFormatter filtra lo que se escribe: rechaza el cambio si no encaja con el patrón
        zipField.setTextFormatter(new TextFormatter<String>(change ->
                change.getControlNewText().matches("[0-9-]{0,10}") ? change : null));
    }

    /**
     * Botón "Sign Up".
     * Empaqueta el formulario en un SignUpRequest, lo envía al servicio y,
     * si todo va bien, informa al usuario y vuelve a Sign In.
     */
    @FXML
    private void handleSignUp() {
        clearError();
        SignUpRequest request = buildRequest();

        Task<User> task = new Task<>() {
            @Override
            protected User call() throws Exception {
                return authService.signUp(request);
            }
        };

        signUpButton.disableProperty().bind(task.runningProperty());

        task.setOnSucceeded(e -> {
            showSuccess(task.getValue());
            SceneNavigator.navigate(signUpButton, Views.SIGN_IN);
        });
        task.setOnFailed(e -> handleError(task.getException()));

        SignInController.startInBackground(task);
    }

    /** Enlace "Sign In": vuelve al inicio de sesión. */
    @FXML
    private void goToSignIn() {
        SceneNavigator.navigate(signInLink, Views.SIGN_IN);
    }

    // ---------------------------------------------------------------- helpers

    /** Lee los controles y crea el objeto de petición. Se lee en el hilo de JavaFX (obligatorio). */
    private SignUpRequest buildRequest() {
        return new SignUpRequest(
                firstNameField.getText(),
                lastNameField.getText(),
                emailField.getText(),
                passwordField.getText(),
                confirmPasswordField.getText(),
                stateField.getText(),
                cityField.getText(),
                streetField.getText(),
                zipField.getText(),
                termsCheckBox.isSelected());
    }

    /** Diálogo de confirmación tras crear la cuenta. */
    private void showSuccess(User user) {
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Cuenta creada");
        alert.setHeaderText("¡Bienvenido, " + user.firstName() + "!");
        alert.setContentText("Tu cuenta se ha creado correctamente. Ya puedes iniciar sesión.");
        alert.initOwner(signUpButton.getScene().getWindow());
        alert.showAndWait();
    }

    /** Muestra el error y resalta el campo correspondiente. */
    private void handleError(Throwable error) {
        if (error instanceof ValidationException ve) {
            FormErrors.show(errorLabel, ve.getMessage());
            Node field = fieldFor(ve.getField());
            if (field != null) FormErrors.highlight(field);
        } else if (error instanceof AuthException ae) {
            FormErrors.show(errorLabel, ae.getMessage());
        } else {
            error.printStackTrace();
            FormErrors.show(errorLabel, "No se pudo crear la cuenta. Inténtalo más tarde.");
        }
    }

    /** Traduce el campo lógico del servicio al control de la interfaz. */
    private Node fieldFor(ValidationException.Field field) {
        return switch (field) {
            case FIRST_NAME -> firstNameField;
            case LAST_NAME -> lastNameField;
            case EMAIL -> emailField;
            case PASSWORD -> passwordField;
            case CONFIRM_PASSWORD -> confirmPasswordField;
            case STATE -> stateField;
            case CITY -> cityField;
            case STREET -> streetField;
            case ZIP_CODE -> zipField;
            case TERMS -> termsCheckBox;
            default -> null;
        };
    }

    private void clearError() {
        FormErrors.clear(errorLabel, allFields());
        FormErrors.clear(errorLabel, termsCheckBox); // La casilla no es un campo de texto, se limpia aparte
    }
}
```

| Método | Disparado por | Qué hace | Por qué así |
|---|---|---|---|
| `initialize()` | JavaFX al cargar | Listeners de limpieza y filtro del código postal | Evita errores de formato antes de enviar |
| `handleSignUp()` | Botón *Sign Up* / Enter | Envía el formulario en segundo plano | El hash tarda; la ventana no se congela |
| `goToSignIn()` | Enlace *Sign In* | Vuelve al login | Navegación coherente |
| `buildRequest()` | Interno | Formulario → `SignUpRequest` | Los controles solo se leen en el hilo de JavaFX |
| `showSuccess(User)` | Registro correcto | Diálogo de bienvenida | Confirmación clara antes de cambiar de vista |
| `handleError(Throwable)` | Fallo del `Task` | Mensaje + campo en rojo | El usuario ve exactamente qué corregir |
| `fieldFor(Field)` | Interno | Enum del servicio → control | El servicio no conoce JavaFX |
| `clearError()` | Escribir / reintentar | Limpia mensaje y bordes | Interfaz limpia |

**¿Por qué no desactivar el botón hasta marcar la casilla de términos?** Se podría
(`signUpButton.disableProperty().bind(termsCheckBox.selectedProperty().not())`), pero un botón
deshabilitado sin explicación confunde. Mostrar "Debes aceptar los términos" es más claro. Además el
botón ya se enlaza a `task.runningProperty()`, y un control solo puede tener un *binding*.

---

### 7.3 `ChangePasswordController`

**`fx:id` usados:** `backButton`, `currentPasswordField`, `newPasswordField`,
`confirmPasswordField`, `errorLabel`, `updatePasswordButton`, `cancelLink`.

```java
package com.example.auth.controller;

import com.example.auth.App;
import com.example.auth.service.AuthException;
import com.example.auth.service.AuthService;
import com.example.auth.service.ValidationException;
import com.example.auth.util.SceneNavigator;
import com.example.auth.util.Session;
import com.example.auth.util.Views;
import javafx.application.Platform;
import javafx.concurrent.Task;
import javafx.fxml.FXML;
import javafx.scene.Node;
import javafx.scene.control.*;

public class ChangePasswordController {

    @FXML private Button backButton;
    @FXML private PasswordField currentPasswordField;
    @FXML private PasswordField newPasswordField;
    @FXML private PasswordField confirmPasswordField;
    @FXML private Label errorLabel;
    @FXML private Button updatePasswordButton;
    @FXML private Hyperlink cancelLink;

    private final AuthService authService = App.authService();

    /**
     * Si no hay sesión (alguien abrió esta vista sin iniciar sesión) se redirige al login.
     * Si la hay, prepara listeners y foco.
     */
    @FXML
    private void initialize() {
        if (!Session.isActive()) {
            // runLater: en initialize() la vista aún no tiene Scene, no se puede navegar todavía
            Platform.runLater(() -> SceneNavigator.navigate(backButton, Views.SIGN_IN));
            return;
        }
        for (PasswordField f : new PasswordField[]{currentPasswordField, newPasswordField, confirmPasswordField}) {
            f.textProperty().addListener((obs, old, val) -> clearError());
        }
        Platform.runLater(currentPasswordField::requestFocus);
    }

    /** Botón "Update Password". */
    @FXML
    private void handleUpdatePassword() {
        clearError();
        String current = currentPasswordField.getText();
        String newPassword = newPasswordField.getText();
        String confirm = confirmPasswordField.getText();

        Task<Void> task = new Task<>() {
            @Override
            protected Void call() throws Exception {
                authService.changePassword(current, newPassword, confirm);
                return null;
            }
        };

        updatePasswordButton.disableProperty().bind(task.runningProperty());

        task.setOnSucceeded(e -> {
            clearFields();
            showSuccess();
            SceneNavigator.navigate(updatePasswordButton, Views.HOME);
        });
        task.setOnFailed(e -> handleError(task.getException()));

        SignInController.startInBackground(task);
    }

    /** Flecha "atrás" y enlace "Cancel": vuelven a la ventana principal sin cambiar nada. */
    @FXML
    private void handleCancel() {
        clearFields();
        SceneNavigator.navigate(cancelLink, Views.HOME);
    }

    // ---------------------------------------------------------------- helpers

    private void showSuccess() {
        Alert alert = new Alert(Alert.AlertType.INFORMATION);
        alert.setTitle("Contraseña actualizada");
        alert.setHeaderText(null);
        alert.setContentText("Tu contraseña se ha cambiado correctamente.");
        alert.initOwner(updatePasswordButton.getScene().getWindow());
        alert.showAndWait();
    }

    private void handleError(Throwable error) {
        if (error instanceof ValidationException ve) {
            FormErrors.show(errorLabel, ve.getMessage());
            Node field = switch (ve.getField()) {
                case CURRENT_PASSWORD -> currentPasswordField;
                case PASSWORD -> newPasswordField;
                case CONFIRM_PASSWORD -> confirmPasswordField;
                default -> null;
            };
            if (field != null) FormErrors.highlight(field);
        } else if (error instanceof AuthException ae) {
            // Sin sesión: se informa y se manda al login
            FormErrors.show(errorLabel, ae.getMessage());
            if (!Session.isActive()) {
                SceneNavigator.navigate(updatePasswordButton, Views.SIGN_IN);
            }
        } else {
            error.printStackTrace();
            FormErrors.show(errorLabel, "No se pudo cambiar la contraseña. Inténtalo más tarde.");
        }
    }

    /** No deja contraseñas escritas en pantalla al salir. */
    private void clearFields() {
        currentPasswordField.clear();
        newPasswordField.clear();
        confirmPasswordField.clear();
    }

    private void clearError() {
        FormErrors.clear(errorLabel, currentPasswordField, newPasswordField, confirmPasswordField);
    }
}
```

| Método | Disparado por | Qué hace | Por qué así |
|---|---|---|---|
| `initialize()` | JavaFX al cargar | Protege la vista si no hay sesión; listeners y foco | Nadie cambia una contraseña sin estar identificado |
| `handleUpdatePassword()` | Botón *Update Password* / Enter | Cambia la contraseña en segundo plano | Verificar y hashear tarda |
| `handleCancel()` | Flecha atrás y *Cancel* | Vuelve a Home sin cambios | Un único método para dos controles equivalentes |
| `showSuccess()` | Cambio correcto | Diálogo de confirmación | El usuario sabe que se aplicó |
| `handleError(Throwable)` | Fallo del `Task` | Mensaje + campo en rojo | Distingue "actual incorrecta" de "no coinciden" |
| `clearFields()` | Éxito / cancelar | Vacía las contraseñas | No dejar datos sensibles visibles |
| `clearError()` | Escribir / reintentar | Limpia mensaje y bordes | Interfaz limpia |

### Cerrar sesión (en tu `HomeController`)

```java
@FXML
private void handleSignOut() {
    App.authService().signOut();
    SceneNavigator.navigate(signOutButton, Views.SIGN_IN);
}

@FXML
private void goToChangePassword() {
    SceneNavigator.navigate(changePasswordButton, Views.CHANGE_PASSWORD);
}
```

---

## 8. Cambios necesarios en los FXML y el CSS

### `fx:controller` y `onAction`

| Archivo | En la etiqueta raíz | Controles |
|---|---|---|
| `sign_in.fxml` | `fx:controller="com.example.auth.controller.SignInController"` | `signInButton` → `onAction="#handleSignIn"`<br>`forgotPasswordLink` → `onAction="#handleForgotPassword"`<br>`signUpLink` → `onAction="#goToSignUp"` |
| `sign_up.fxml` | `fx:controller="com.example.auth.controller.SignUpController"` | `signUpButton` → `onAction="#handleSignUp"`<br>`signInLink` → `onAction="#goToSignIn"` |
| `change_password.fxml` | `fx:controller="com.example.auth.controller.ChangePasswordController"` | `updatePasswordButton` → `onAction="#handleUpdatePassword"`<br>`backButton` → `onAction="#handleCancel"`<br>`cancelLink` → `onAction="#handleCancel"` |

Si en `sign_up.fxml` usas un `ScrollPane` como raíz, el `fx:controller` va en el `ScrollPane`.

Ejemplo:

```xml
<Button fx:id="signInButton" styleClass="auth-button" text="Sign In"
        maxWidth="Infinity" defaultButton="true" onAction="#handleSignIn"/>
```

### Añadir `errorLabel` a Sign In y Sign Up

`change_password.fxml` ya lo tiene. En los otros dos, pégalo dentro de la tarjeta, **justo encima del
botón principal**:

```xml
<Label fx:id="errorLabel" styleClass="auth-error" wrapText="true"
       managed="false" visible="false"/>
```

### Borde rojo en campos con error (`auth.css`)

```css
.auth-input.error {
    -fx-border-color: #d93025;
    -fx-background-color: #fdf3f2;
}

.auth-checkbox.error {
    -fx-text-fill: #d93025;
}
```

`FormErrors.highlight(...)` añade la clase `error` al `HBox .auth-input` del campo (o a la casilla de
términos), y `FormErrors.clear(...)` la quita.

---

## 9. Resumen de métodos

| Clase | Método | Devuelve | Lanza |
|---|---|---|---|
| `AuthService` | `signIn(email, password)` | `User` | `ValidationException`, `AuthException` |
| | `signUp(SignUpRequest)` | `User` | `ValidationException`, `AuthException` |
| | `changePassword(current, new, confirm)` | `void` | `ValidationException`, `AuthException` |
| | `signOut()` | `void` | — |
| `UserRepository` | `findByEmail(email)` | `Optional<User>` | `IllegalStateException` (BD) |
| | `findById(id)` | `Optional<User>` | `IllegalStateException` (BD) |
| | `existsByEmail(email)` | `boolean` | `IllegalStateException` (BD) |
| | `save(User)` | `User` con id | `IllegalStateException` (BD) |
| | `updatePassword(id, hash)` | `boolean` | `IllegalStateException` (BD) |
| `PasswordHasher` | `hash(password)` | `String` | — |
| | `verify(password, stored)` | `boolean` | — |
| `Validator` | `isBlank`, `isValidEmail`, `isStrongPassword`, `isValidZip`, `normalizeEmail` | `boolean` / `String` | — |
| `Session` | `start`, `end`, `getCurrentUser`, `isActive`, `refresh` | — | — |
| `SceneNavigator` | `navigate(source, fxml)` | `void` | `IllegalArgumentException`, `UncheckedIOException` |
| `SignInController` | `initialize`, `handleSignIn`, `handleForgotPassword`, `goToSignUp` | `void` | — |
| `SignUpController` | `initialize`, `handleSignUp`, `goToSignIn` | `void` | — |
| `ChangePasswordController` | `initialize`, `handleUpdatePassword`, `handleCancel` | `void` | — |

---

## 10. Cómo probarlo

### Pruebas manuales (lista para la demo)

| # | Pantalla | Acción | Resultado esperado |
|---|---|---|---|
| 1 | Sign Up | Enviar vacío | "Introduce tu nombre.", First Name en rojo |
| 2 | Sign Up | Email `ana@` | "El email no tiene un formato válido." |
| 3 | Sign Up | Contraseña `abc` | Mensaje de mínimo 8 caracteres con letras y números |
| 4 | Sign Up | Confirmación distinta | "Las contraseñas no coinciden.", Confirm en rojo |
| 5 | Sign Up | Zip `123` | "El código postal debe tener 5 dígitos." |
| 6 | Sign Up | Sin marcar términos | Mensaje de términos, casilla en rojo |
| 7 | Sign Up | Todo correcto | Diálogo de bienvenida → vuelve a Sign In |
| 8 | Sign Up | Repetir el mismo email | "Ya existe una cuenta con ese email." |
| 9 | Sign In | Contraseña incorrecta | "Email o contraseña incorrectos.", contraseña borrada |
| 10 | Sign In | `ANA@x.com` (mayúsculas) | Entra igual (email normalizado) |
| 11 | Change Password | Actual incorrecta | "La contraseña actual no es correcta." |
| 12 | Change Password | Nueva = actual | "La nueva contraseña debe ser distinta de la actual." |
| 13 | Change Password | Todo correcto, cerrar sesión y entrar con la nueva | Funciona; la antigua ya no |

### Prueba automática de ejemplo (JUnit 5)

```java
class AuthServiceTest {

    private AuthService service;

    @BeforeEach
    void setUp() {
        service = new AuthService(new InMemoryUserRepository()); // Sin BD ni ventanas
        Session.end();
    }

    private SignUpRequest validRequest() {
        return new SignUpRequest("Ana", "López", "ana@test.com", "secreta123", "secreta123",
                "Texas", "Austin", "123 Main St", "73301", true);
    }

    @Test
    void signUpThenSignInWorks() throws Exception {
        service.signUp(validRequest());
        User user = service.signIn("ANA@test.com", "secreta123");
        assertEquals("Ana", user.firstName());
        assertTrue(Session.isActive());
    }

    @Test
    void wrongPasswordIsRejected() throws Exception {
        service.signUp(validRequest());
        assertThrows(AuthException.class, () -> service.signIn("ana@test.com", "otra12345"));
    }

    @Test
    void duplicateEmailIsRejected() throws Exception {
        service.signUp(validRequest());
        ValidationException ex = assertThrows(ValidationException.class, () -> service.signUp(validRequest()));
        assertEquals(ValidationException.Field.EMAIL, ex.getField());
    }

    @Test
    void changePasswordReplacesOldOne() throws Exception {
        service.signUp(validRequest());
        service.signIn("ana@test.com", "secreta123");
        service.changePassword("secreta123", "nueva4567", "nueva4567");
        service.signOut();

        assertThrows(AuthException.class, () -> service.signIn("ana@test.com", "secreta123"));
        assertNotNull(service.signIn("ana@test.com", "nueva4567"));
    }
}
```

**Argumento final:** gracias a la separación en capas, estas pruebas validan **toda la lógica de
autenticación** en milisegundos, sin base de datos ni interfaz. Si fallan, el problema está en el
backend; si pasan y algo va mal en pantalla, el problema está en el controlador o el FXML.
