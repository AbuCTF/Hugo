---
title: "ClassFlow"
description: "QR-Based Attendance Tracking"
icon: "article"
date: "2024-03-15"
lastmod: "2024-03-15"
draft: false
toc: true
weight: 998
---

Ah, I see a man of culture : )

{{< figure src="cover.jpg" alt="ClassFlow Cover" >}}

This is the documentation for ClassFlow, a QR-based attendance tracking application built with Java and PostgreSQL. The project was developed as a practical solution to the age-old problem of taking attendance in large classrooms.

### Project Structure

The directory structure follows standard Android/Java application conventions with clear separation of concerns.

```
ClassFlow/
├── app/
│   ├── AttendanceTrackerApp.java
│   └── Main.java
├── qr/
│   ├── QRCodeScanner.java
│   ├── QRCodeGenerator.java
│   └── QRCodeUtil.java
├── user/
│   ├── User.java
│   └── UserManager.java
├── attendance/
│   ├── AttendanceRecord.java
│   ├── AttendanceManager.java
│   ├── AttendanceService.java
│   └── AttendanceReportGenerator.java
├── data/
│   ├── DatabaseHandler.java
│   └── AttendanceDatabase.java
├── ui/
│   ├── activities/
│   ├── fragments/
│   ├── adapters/
│   ├── dialog/
│   └── utilities/
├── auth/
│   ├── AuthManager.java
│   ├── AuthHandler.java
│   └── AuthResponse.java
├── network/
│   ├── ApiClient.java
│   └── ApiService.java
└── shared/
    ├── Constants.java
    ├── AppConfig.java
    └── CrossPlatformUtilities.java
```

Each package serves a specific purpose:

1. **app**: Main application class and entry point
2. **qr**: QR code scanning, generation, and utilities
3. **user**: User entity and management operations
4. **attendance**: Record keeping, management, and reporting
5. **data**: Database connection and schema management
6. **ui**: Activities, fragments, adapters, and UI utilities
7. **auth**: Authentication and authorization handling
8. **network**: API client and service definitions
9. **shared**: Constants, configuration, and cross-platform utilities

### Database Implementation

PostgreSQL is used as the backend database. The `data` package handles all database interactions through network requests since PostgreSQL is server-based.

`DatabaseHandler.java` manages the connection setup, pooling, and lifecycle. It establishes connections to the PostgreSQL server and handles reconnection logic.

`AttendanceDatabase.java` defines the schema and SQL queries for creating tables, inserting records, and querying data. The PostgreSQL JDBC driver facilitates communication between the Android app and the server.

```java
// Example connection setup
public class DatabaseConnector {
    private static final String URL = "jdbc:postgresql://localhost:5432/classflow";
    private static final String USER = "classflow_user";
    
    public Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, getPassword());
    }
}
```

### Encryption

The application uses AES (Advanced Encryption Standard) for securing sensitive data. The `Cipher` class from Java Cryptography Extension handles encryption and decryption operations.

{{< figure src="diagram.png" alt="ClassFlow Diagram" >}}

The encryption flow:

1. **Key Generation**: Secret key loaded from secure storage
2. **Cipher Initialization**: Create instance with `Cipher.ENCRYPT_MODE` and the secret key
3. **Data Processing**: Input data transformed in blocks using AES algorithm
4. **Padding**: PKCS5Padding ensures data fits 128-bit block size
5. **Output**: Encrypted data used for QR code UUID generation

```java
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, secretKey);
byte[] encrypted = cipher.doFinal(data.getBytes());
```

Password hashing uses BCrypt with proper salting to prevent rainbow table attacks.

### QR Code System

The QR code workflow is straightforward:

1. **Generation (Teacher)**: Teacher starts a session, system generates QR code containing session ID and encrypted UUID
2. **Display**: QR code shown on screen or printed
3. **Scanning (Student)**: Student scans with app, camera processes the code
4. **Extraction**: App extracts session ID and UUID from QR content
5. **Verification**: System validates UUID against stored session info, checks timestamp and duration
6. **Recording**: If valid, attendance record created with user ID, timestamp, and session ID

The `session_info.txt` file stores session metadata:

```makefile
Session1:UUID1:1699123456:3600
Session2:UUID2:1699127056:3600
Session3:UUID3:1699130656:3600
```

Each line contains: Session ID, UUID, start timestamp, and duration in seconds.

### User Management

User registration implements several security measures:

```java
public boolean registerUser(String username, String password, int maxUsers) {
    if (countRegisteredUsers() >= maxUsers) {
        return false; // Registration limit reached
    }
    String hashedPassword = hashAndSaltPassword(password);
    return insertUserIntoDatabase(username, hashedPassword);
}
```

Login verification:

```java
public boolean loginUser(String username, String password) {
    String storedHash = getHashedPasswordFromDatabase(username);
    if (storedHash == null) return false;
    return BCrypt.checkpw(password, storedHash);
}
```

Input validation enforces:
- Minimum 8 character passwords
- Mix of uppercase, lowercase, numbers, and special characters
- Check against common password dictionary

### User Interface

{{< figure src="screenshot.jpg" alt="ClassFlow UI" >}}

The UI follows MVC (Model-View-Controller) architecture with JavaFX:

- **Model**: Data and business logic (`User`, `AttendanceRecord`)
- **View**: FXML layouts defining UI structure
- **Controller**: Java classes handling user interactions

```
ui/
├── controllers/
│   ├── LoginController.java
│   ├── DashboardController.java
│   └── AttendanceController.java
├── fragments/
├── dialogs/
├── adapters/
└── layouts/
    ├── login.fxml
    ├── dashboard.fxml
    └── attendance.fxml
```

Running the JavaFX application:

```bash
mvn clean javafx:run
```

### Authentication

The auth package implements token-based authentication:

**AuthManager.java**: Handles login/logout, manages authentication state
```java
public void login(String username, String password) {
    if (UserLoginManager.verify(username, password)) {
        String token = AuthHandler.generateToken(username);
        setAuthToken(token);
        setLoggedIn(true);
    }
}
```

**AuthHandler.java**: Token generation, validation, and verification using JWT

**AuthResponse.java**: Represents authentication response with token and user info

The flow:
1. `UserLoginManager` verifies credentials
2. `AuthHandler` generates JWT token
3. `AuthManager` stores token and updates login status

### UUID Generation

Each QR code contains a unique identifier generated from:
- Session ID
- Timestamp
- Counter value
- Secret key encryption

```java
String combined = sessionId + ":" + timestamp + ":" + counter;
byte[] encrypted = encryptWithAES(combined, secretKey);
String uuid = UUID.nameUUIDFromBytes(encrypted).toString();
```

Uses:
- Record keeping and audit trails
- Session association and tracking
- Authentication and validity verification

### Role-Based Access

Different user roles have different capabilities:
- **Teachers**: QR code generation
- **Students**: QR code scanning
- **Admins**: Data retrieval and management

The users table includes a role column:

```sql
ALTER TABLE users ADD COLUMN role VARCHAR(20) DEFAULT 'student';
```

### Development Log

**#01**
- Username validity (no redundancy)
- User limit indication
- Password input logic
- Common passwords file integration

**#02**
- GUI implementation in ui package
- Delete user function
- QR package code updates
- Security package for admin key generation

**#02.1**
- Simple GUI for registration and login with pop-ups
- JavaFX dependencies in pom.xml
- gui sub-package under ui

**#02.2**
- GUI updated for email in registration
- Delete function with built-in admin password

**#03**
- QR package integration in main()
- Feature access only for logged users
- Teacher-only QR generation
- Database connection for all functions

**#03.1**
- Login/logout feature in UserLoginManager
- AuthHandler & AuthManager for JWT tokens

**#04.1**
- QR package called after login verification
- QR path input for saving images

**#05.1**
- UserMenuGUI for secondary portal
- Tested complete GUI actions
- Roles implementation in application
- PostgreSQL column for user roles

**#06.1**
- AttendanceManager INSERT after scan verification
- user_id creation based on username prefix + UUID

**#07.1**
- QR verification with UUID + session ID
- Scanner passes control to attendance package on success
- Multiple valid QRs support in config files
- Role-based feature segregation
- Logging framework consideration (Log4j, SLF4J)

---

Built as a college project, turned into something actually useful.
