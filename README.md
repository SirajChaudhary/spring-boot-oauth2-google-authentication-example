# Spring Boot Google OAuth2 Login Example

---

## About This Project

This is a **Spring Boot application** demonstrating **Google OAuth2 login integration**.

After a successful Google login, the app:

- Retrieves a **token from Google** which contains the user’s details (such as email, name, given_name, family_name, and unique ID).
- Uses a **callback API** `/google/success` called by Spring Security after login.
- Generates a **custom JWT token** containing the user’s information and roles.
- Secures **REST APIs** with token-based access so that only authenticated users can access protected endpoints.

### How It Works

1. The user logs in with **Google OAuth2**.
2. Google sends back an **ID token in JWT format**, which contains user information such as:
    - **email** → user’s email address
    - **name** → full name
    - **given_name** → first name
    - **family_name** → last name
    - **sub** → unique Google user ID (can be used as username)
3. After a successful login, Spring Security calls the **callback API** `/google/success`.
4. The app reads the ID token and extracts the user’s **claims** (email, name, given_name, family_name, username).
5. The app then generates a **custom JWT token** including the user’s information and roles.
6. This **custom JWT token** is returned to the user and can be used to access the app’s **protected APIs**.

### Benefits of Using a Custom JWT

- Include **roles and permissions** for role-based access control.
- Security becomes **independent of Google** or any other OAuth provider.
- Works for **multiple providers** if you add GitHub, Facebook, etc.
- Makes APIs **stateless** and scalable.

**Note:** Currently, the app works for **one user at a time**. To support **many users with different roles**, you need to integrate a **database** and implement user/role checks (see below).

---

## Google OAuth2 Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials).  
2. Create OAuth 2.0 Client ID → Web application.  
3. Set Authorized redirect URI: http://localhost:8080/login/oauth2/code/google  
4. Copy the Client ID and Client Secret.  
5. Update application.properties:

```properties
spring.security.oauth2.client.registration.google.client-id=YOUR_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_CLIENT_SECRET
spring.security.oauth2.client.registration.google.scope=openid,profile,email
spring.security.oauth2.client.registration.google.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
```

---

## Running the Application

1. Clone this repository.  
2. Configure application.properties with your Google credentials.  
3. Run the app:

```bash
mvn spring-boot:run
```

4. Open browser → login with Google:

```
http://localhost:8080/oauth2/authorization/google
```

5. Open Swagger UI: http://localhost:8080/swagger-ui.html → explore secured APIs.

---

## Do We Need a Custom JWT?

- Without custom JWT: You rely directly on Google’s ID token. Good for simple setups.  
- With custom JWT: Issue your own JWT after login. Benefits:
  - Works across providers (Google, GitHub, etc.)  
  - Add roles, permissions, internal claims  
  - Consistent API security independent of external providers  

Recommended: Generate a custom JWT if your system needs RBAC (Role-Based Access Control) or integration with internal databases.

---

## How to convert this App to support multiple Users & Roles

**Current Scenario:**  
- Single Google login session.  
- No persistent user store.  
- All users have default access; no role distinction.  

**Why Multi-User & Role Support is Needed:**  
- Track multiple users in a database.  
- Assign roles (USER, ADMIN) for role-based API access.  
- Include roles in custom JWT for consistent and secure API access.  
- Make the app scalable and production-ready with RBAC.  

### 1. Database Setup

```sql
CREATE DATABASE spring_google_oauth2_example;
USE spring_google_oauth2_example;

CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE, -- Google email
    name VARCHAR(255) NOT NULL,
    roles VARCHAR(255) NOT NULL           -- e.g., "ROLE_USER,ROLE_ADMIN"
);

-- Optional: insert an ADMIN user
INSERT INTO users (username, name, roles)
VALUES ('SirajChaudhary', 'Siraj Chaudhary', 'ROLE_ADMIN');
```

### 2. User Entity

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "users")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable=false, unique=true)
    private String username; // Google email

    @Column(nullable=false)
    private String name;

    @Column(nullable=false)
    private String roles; // e.g., "ROLE_USER,ROLE_ADMIN"
}
```

### 3. User Repository

```java
package com.example.demo.repository;

import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

### 4. AuthController Update (Google Login Success)

```java
@GetMapping("/success")
public Map<String, Object> loginSuccess(@AuthenticationPrincipal OAuth2User principal) {
    String email = principal.getAttribute("email");
    String name = principal.getAttribute("name");

    // Save user if not exists
    User user = userRepository.findByUsername(email)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setUsername(email);
                newUser.setName(name);
                newUser.setRoles("ROLE_USER"); // default role
                return userRepository.save(newUser);
            });

    // Generate JWT with roles from DB
    Map<String, Object> claims = new HashMap<>();
    claims.put("name", user.getName());
    claims.put("roles", user.getRoles());

    String token = jwtUtil.generateToken(claims, user.getUsername());

    return Map.of(
            "message", "Login successful",
            "jwt", token,
            "user", user
    );
}
```

### 5. Role-Based Endpoint Example

```java
@RestController
@RequestMapping("/api/v1/employees")
public class EmployeeController {

    @GetMapping("/user/hello")
    @PreAuthorize("hasRole('USER')")
    public String userHello() {
        return "Hello User! You have USER access.";
    }

    @GetMapping("/admin/hello")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminHello() {
        return "Hello Admin! You have ADMIN access.";
    }
}
```

### 6. Security Configuration

```java
@EnableMethodSecurity // enables @PreAuthorize
@Configuration
public class SecurityConfig {
    // existing OAuth2 + JWT configuration
}
```

---

## Notes

- Current app supports one stateless JWT per login.  
- Multi-user RBAC requires database integration (users + roles).  
- Custom JWT helps standardize API security, add roles/permissions, and integrate with internal databases.  
- Role-based access can be applied to endpoints using `@PreAuthorize("hasRole('ROLE_NAME')")`.

---

## License

Free Software, by Siraj Chaudhary