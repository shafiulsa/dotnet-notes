(Angular & React)** projects, along with the **CORS configuration** to avoid errors.

---

## 🏗️ Backend Folder Structure (ASP.NET Core Web API)

```
CompliteAuthTestingApp/
│
├── Controllers/
│    ├── AuthController.cs
│    ├── RoleTestingController.cs
│    └── ... other controllers ...
│
├── Data/
│    ├── AppDbContext.cs
│    └── SeedRole.cs
│
├── Extensions/
│    └── JwtServiceExtensions.cs  // extension method for JWT + Swagger
│
├── Models/
│    ├── ApplicationUser.cs
│    ├── RegisterDto.cs
│    ├── LoginDto.cs
│    └── SetUserRoleRequest.cs
│
├── Services/
│    └── TokenService.cs           // creates JWT tokens
│
├── program.cs
├── appsettings.json
└── CompliteAuthTestingApp.csproj
```

---

## ⚙️ Backend CORS Configuration in program.cs

Make sure you add CORS to allow your frontend URLs, **before** authentication/authorization middleware:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add CORS policy
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("http://localhost:4200", "http://localhost:3000") // Angular and React dev servers
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

// Add services...
builder.Services.AddControllers();
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequireNonAlphanumeric = false;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

builder.Services.AddJwtAndSwagger(builder.Configuration);

var app = builder.Build();

// Use CORS before authentication & authorization
app.UseCors("AllowFrontend");

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---





## 🎯 How to avoid CORS errors

* Make sure the **backend `program.cs` uses `app.UseCors("AllowFrontend")` BEFORE `app.UseAuthentication()`**.
* The origins in `WithOrigins(...)` must match your frontend app URL exactly, including protocol and port.
* For development, typical URLs are:

  * Angular: `http://localhost:4200`
  * React: `http://localhost:3000`
* When you deploy, update the allowed origins accordingly.

---

## Example:

Angular frontend runs at:
`http://localhost:4200`

React frontend runs at:
`http://localhost:3000`

Backend CORS setup:

```csharp
policy.WithOrigins("http://localhost:4200", "http://localhost:3000")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

---







<br>
<br>
<br>


# React JWT + Role-Based Auth Example

---

## 1️⃣ React Folder Structure

```
my-react-app/
│
├── public/
│   └── index.html
│
├── src/
│   ├── api/
│   │   └── authApi.js          // API calls (login + role endpoints)
│   │
│   ├── components/
│   │   ├── Login.js            // Login form & logic
│   │   ├── AdminPanel.js       // Admin role example component
│   │   ├── UserPanel.js        // User role example component
│   │   ├── StaffPanel.js       // Staff role example component
│   │   └── ModeratorPanel.js   // Moderator role example component
│   │
│   ├── App.js                  // Main app router + role-based rendering
│   └── index.js
│
├── package.json
└── README.md
```

---

## 2️⃣ Code with comments

### src/api/authApi.js

```js
// authApi.js - central place for API calls

const API_BASE_URL = "https://your-api-url"; // Replace with your backend URL

// Save and get JWT token from localStorage
export const saveToken = (token) => localStorage.setItem("jwt_token", token);
export const getToken = () => localStorage.getItem("jwt_token");

// Login user and get JWT token
export async function login(email, password) {
  const res = await fetch(`${API_BASE_URL}/api/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });

  if (!res.ok) {
    throw new Error("Login failed");
  }

  const data = await res.json();
  saveToken(data.token);
  return data.token;
}

// Helper to add Authorization header with token
function authHeader() {
  return { Authorization: "Bearer " + getToken() };
}

// API calls for each role protected endpoint

export async function fetchAdminData() {
  const res = await fetch(`${API_BASE_URL}/api/roletesting/admin-only`, {
    headers: { ...authHeader() },
  });
  if (!res.ok) throw new Error("Unauthorized or error");
  return res.json();
}

export async function fetchUserData() {
  const res = await fetch(`${API_BASE_URL}/api/roletesting/user-only`, {
    headers: { ...authHeader() },
  });
  if (!res.ok) throw new Error("Unauthorized or error");
  return res.json();
}

export async function fetchStaffData() {
  const res = await fetch(`${API_BASE_URL}/api/roletesting/staff-only`, {
    headers: { ...authHeader() },
  });
  if (!res.ok) throw new Error("Unauthorized or error");
  return res.json();
}

export async function fetchModeratorData() {
  const res = await fetch(`${API_BASE_URL}/api/roletesting/moderator-only`, {
    headers: { ...authHeader() },
  });
  if (!res.ok) throw new Error("Unauthorized or error");
  return res.json();
}
```

---

### src/components/Login.js

```jsx
import React, { useState } from "react";
import { login } from "../api/authApi";

export default function Login({ onLoginSuccess }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(e) {
    e.preventDefault();
    setError("");
    try {
      await login(email, password);
      onLoginSuccess();
    } catch (err) {
      setError("Login failed. Check your credentials.");
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Login</h2>
      <input
        type="email"
        placeholder="Email"
        required
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      /><br />
      <input
        type="password"
        placeholder="Password"
        required
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      /><br />
      <button type="submit">Login</button>
      {error && <p style={{ color: "red" }}>{error}</p>}
    </form>
  );
}
```

---

### src/components/AdminPanel.js

```jsx
import React, { useEffect, useState } from "react";
import { fetchAdminData } from "../api/authApi";

export default function AdminPanel() {
  const [message, setMessage] = useState("");

  useEffect(() => {
    fetchAdminData()
      .then((data) => setMessage(JSON.stringify(data)))
      .catch(() => setMessage("You are not authorized or error occurred."));
  }, []);

  return (
    <div>
      <h3>Admin Panel</h3>
      <pre>{message}</pre>
    </div>
  );
}
```

---

### src/components/UserPanel.js

```jsx
import React, { useEffect, useState } from "react";
import { fetchUserData } from "../api/authApi";

export default function UserPanel() {
  const [message, setMessage] = useState("");

  useEffect(() => {
    fetchUserData()
      .then((data) => setMessage(JSON.stringify(data)))
      .catch(() => setMessage("You are not authorized or error occurred."));
  }, []);

  return (
    <div>
      <h3>User Panel</h3>
      <pre>{message}</pre>
    </div>
  );
}
```

---

### src/components/StaffPanel.js

```jsx
import React, { useEffect, useState } from "react";
import { fetchStaffData } from "../api/authApi";

export default function StaffPanel() {
  const [message, setMessage] = useState("");

  useEffect(() => {
    fetchStaffData()
      .then((data) => setMessage(JSON.stringify(data)))
      .catch(() => setMessage("You are not authorized or error occurred."));
  }, []);

  return (
    <div>
      <h3>Staff Panel</h3>
      <pre>{message}</pre>
    </div>
  );
}
```

---

### src/components/ModeratorPanel.js

```jsx
import React, { useEffect, useState } from "react";
import { fetchModeratorData } from "../api/authApi";

export default function ModeratorPanel() {
  const [message, setMessage] = useState("");

  useEffect(() => {
    fetchModeratorData()
      .then((data) => setMessage(JSON.stringify(data)))
      .catch(() => setMessage("You are not authorized or error occurred."));
  }, []);

  return (
    <div>
      <h3>Moderator Panel</h3>
      <pre>{message}</pre>
    </div>
  );
}
```

---

### src/App.js

```jsx
import React, { useState, useEffect } from "react";
import Login from "./components/Login";
import AdminPanel from "./components/AdminPanel";
import UserPanel from "./components/UserPanel";
import StaffPanel from "./components/StaffPanel";
import ModeratorPanel from "./components/ModeratorPanel";

// Helper to parse JWT token and get roles
function parseJwt(token) {
  if (!token) return null;
  try {
    const base64Payload = token.split('.')[1];
    const payload = atob(base64Payload);
    return JSON.parse(payload);
  } catch {
    return null;
  }
}

export default function App() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const [roles, setRoles] = useState([]);

  useEffect(() => {
    const token = localStorage.getItem("jwt_token");
    if (token) {
      const parsed = parseJwt(token);
      if (parsed && parsed.role) {
        setRoles(Array.isArray(parsed.role) ? parsed.role : [parsed.role]);
        setIsLoggedIn(true);
      }
    }
  }, []);

  if (!isLoggedIn) {
    return <Login onLoginSuccess={() => window.location.reload()} />;
  }

  return (
    <div>
      <h1>Role Based Panels</h1>

      {/* Render panel if user has role */}
      {roles.includes("Admin") && <AdminPanel />}
      {roles.includes("User") && <UserPanel />}
      {roles.includes("Staff") && <StaffPanel />}
      {roles.includes("Moderator") && <ModeratorPanel />}

      <button onClick={() => {
        localStorage.removeItem("jwt_token");
        window.location.reload();
      }}>
        Logout
      </button>
    </div>
  );
}
```

---

## Summary:

* Login sets JWT token in localStorage
* `App.js` parses JWT, reads roles, and shows panels based on roles
* Each panel makes a fetch call to role-protected API endpoint using the stored JWT token in Authorization header
* If unauthorized, an error message is shown

---


Sure! Here’s a clear Angular setup with:

* Folder structure
* Login and JWT token storage
* HTTP Interceptor to add token to requests
* One component/service per role (Admin, User, Staff, Moderator) making calls to your protected backend endpoints
* Proper comments for clarity

---

# Angular JWT + Role-Based Auth Example

---

## 1️⃣ Angular Folder Structure

```
my-angular-app/
│
├── src/
│   ├── app/
│   │   ├── services/
│   │   │   ├── auth.service.ts
│   │   │   ├── role.service.ts
│   │   │   └── auth.interceptor.ts
│   │   ├── components/
│   │   │   ├── login/
│   │   │   │   ├── login.component.ts
│   │   │   │   └── login.component.html
│   │   │   ├── admin-panel/
│   │   │   │   ├── admin-panel.component.ts
│   │   │   │   └── admin-panel.component.html
│   │   │   ├── user-panel/
│   │   │   │   ├── user-panel.component.ts
│   │   │   │   └── user-panel.component.html
│   │   │   ├── staff-panel/
│   │   │   │   ├── staff-panel.component.ts
│   │   │   │   └── staff-panel.component.html
│   │   │   └── moderator-panel/
│   │   │       ├── moderator-panel.component.ts
│   │   │       └── moderator-panel.component.html
│   │   ├── app.component.ts
│   │   └── app.module.ts
│   └── main.ts
├── angular.json
├── package.json
└── tsconfig.json
```

---

## 2️⃣ Core Code Files

---

### src/app/services/auth.service.ts

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { BehaviorSubject, tap } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private baseUrl = 'https://your-api-url/api/auth'; // Replace with your backend URL

  private loggedIn = new BehaviorSubject<boolean>(this.hasToken());
  loggedIn$ = this.loggedIn.asObservable();

  constructor(private http: HttpClient) {}

  // Check if token exists in localStorage
  private hasToken(): boolean {
    return !!localStorage.getItem('jwt_token');
  }

  // Login API call
  login(email: string, password: string) {
    return this.http.post<{ token: string }>(`${this.baseUrl}/login`, { email, password })
      .pipe(
        tap(res => {
          localStorage.setItem('jwt_token', res.token);
          this.loggedIn.next(true);
        })
      );
  }

  // Logout user and clear token
  logout() {
    localStorage.removeItem('jwt_token');
    this.loggedIn.next(false);
  }

  // Get JWT token for interceptor usage
  getToken(): string | null {
    return localStorage.getItem('jwt_token');
  }
}
```

---

### src/app/services/auth.interceptor.ts

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable } from 'rxjs';
import { AuthService } from './auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();

    if (token) {
      // Clone request and add Authorization header
      const cloned = req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      });
      return next.handle(cloned);
    }

    return next.handle(req);
  }
}
```

Register this interceptor in `app.module.ts` providers:

```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { AuthInterceptor } from './services/auth.interceptor';

@NgModule({
  // ...
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ]
})
export class AppModule {}
```

---

### src/app/services/role.service.ts

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class RoleService {
  private baseUrl = 'https://your-api-url/api/roletesting'; // Replace with your backend URL

  constructor(private http: HttpClient) {}

  getAdminData() {
    return this.http.get(`${this.baseUrl}/admin-only`);
  }

  getUserData() {
    return this.http.get(`${this.baseUrl}/user-only`);
  }

  getStaffData() {
    return this.http.get(`${this.baseUrl}/staff-only`);
  }

  getModeratorData() {
    return this.http.get(`${this.baseUrl}/moderator-only`);
  }
}
```

---

### src/app/components/login/login.component.ts

```ts
import { Component } from '@angular/core';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  templateUrl: './login.component.html'
})
export class LoginComponent {
  email = '';
  password = '';
  errorMessage = '';

  constructor(private authService: AuthService) {}

  login() {
    this.errorMessage = '';
    this.authService.login(this.email, this.password).subscribe({
      next: () => {
        // On successful login reload or navigate
        window.location.reload();
      },
      error: () => {
        this.errorMessage = 'Login failed. Please check credentials.';
      }
    });
  }
}
```

---

### src/app/components/login/login.component.html

```html
<h2>Login</h2>
<form (submit)="login(); $event.preventDefault()">
  <input type="email" placeholder="Email" [(ngModel)]="email" name="email" required /><br />
  <input type="password" placeholder="Password" [(ngModel)]="password" name="password" required /><br />
  <button type="submit">Login</button>
  <p style="color:red" *ngIf="errorMessage">{{ errorMessage }}</p>
</form>
```

---

### src/app/components/admin-panel/admin-panel.component.ts

```ts
import { Component, OnInit } from '@angular/core';
import { RoleService } from '../../services/role.service';

@Component({
  selector: 'app-admin-panel',
  templateUrl: './admin-panel.component.html'
})
export class AdminPanelComponent implements OnInit {
  message = '';

  constructor(private roleService: RoleService) {}

  ngOnInit() {
    this.roleService.getAdminData().subscribe({
      next: (data) => this.message = JSON.stringify(data),
      error: () => this.message = 'Unauthorized or error occurred'
    });
  }
}
```

---

### src/app/components/admin-panel/admin-panel.component.html

```html
<h3>Admin Panel</h3>
<pre>{{ message }}</pre>
```

---

### src/app/components/user-panel/user-panel.component.ts

```ts
import { Component, OnInit } from '@angular/core';
import { RoleService } from '../../services/role.service';

@Component({
  selector: 'app-user-panel',
  templateUrl: './user-panel.component.html'
})
export class UserPanelComponent implements OnInit {
  message = '';

  constructor(private roleService: RoleService) {}

  ngOnInit() {
    this.roleService.getUserData().subscribe({
      next: (data) => this.message = JSON.stringify(data),
      error: () => this.message = 'Unauthorized or error occurred'
    });
  }
}
```

---

### src/app/components/user-panel/user-panel.component.html

```html
<h3>User Panel</h3>
<pre>{{ message }}</pre>
```

---

### src/app/components/staff-panel/staff-panel.component.ts

```ts
import { Component, OnInit } from '@angular/core';
import { RoleService } from '../../services/role.service';

@Component({
  selector: 'app-staff-panel',
  templateUrl: './staff-panel.component.html'
})
export class StaffPanelComponent implements OnInit {
  message = '';

  constructor(private roleService: RoleService) {}

  ngOnInit() {
    this.roleService.getStaffData().subscribe({
      next: (data) => this.message = JSON.stringify(data),
      error: () => this.message = 'Unauthorized or error occurred'
    });
  }
}
```

---

### src/app/components/staff-panel/staff-panel.component.html

```html
<h3>Staff Panel</h3>
<pre>{{ message }}</pre>
```

---

### src/app/components/moderator-panel/moderator-panel.component.ts

```ts
import { Component, OnInit } from '@angular/core';
import { RoleService } from '../../services/role.service';

@Component({
  selector: 'app-moderator-panel',
  templateUrl: './moderator-panel.component.html'
})
export class ModeratorPanelComponent implements OnInit {
  message = '';

  constructor(private roleService: RoleService) {}

  ngOnInit() {
    this.roleService.getModeratorData().subscribe({
      next: (data) => this.message = JSON.stringify(data),
      error: () => this.message = 'Unauthorized or error occurred'
    });
  }
}
```

---

### src/app/components/moderator-panel/moderator-panel.component.html

```html
<h3>Moderator Panel</h3>
<pre>{{ message }}</pre>
```

---

### src/app/app.component.ts

```ts
import { Component, OnInit } from '@angular/core';
import { AuthService } from './services/auth.service';

@Component({
  selector: 'app-root',
  template: `
    <div *ngIf="!isLoggedIn; else loggedInTemplate">
      <app-login></app-login>
    </div>

    <ng-template #loggedInTemplate>
      <h1>Role Based Panels</h1>
      <button (click)="logout()">Logout</button>

      <app-admin-panel *ngIf="roles.includes('Admin')"></app-admin-panel>
      <app-user-panel *ngIf="roles.includes('User')"></app-user-panel>
      <app-staff-panel *ngIf="roles.includes('Staff')"></app-staff-panel>
      <app-moderator-panel *ngIf="roles.includes('Moderator')"></app-moderator-panel>
    </ng-template>
  `
})
export class AppComponent implements OnInit {
  isLoggedIn = false;
  roles: string[] = [];

  ngOnInit() {
    const token = localStorage.getItem('jwt_token');
    if (token) {
      this.isLoggedIn = true;
      this.roles = this.parseJwtRoles(token);
    }
  }

  constructor(private authService: AuthService) {}

  logout() {
    this.authService.logout();
    window.location.reload();
  }

  // Parses JWT token payload and extracts roles (assumes roles are in 'role' claim)
  parseJwtRoles(token: string): string[] {
    try {
      const base64Payload = token.split('.')[1];
      const payload = atob(base64Payload);
      const parsed = JSON.parse(payload);
      const roles = parsed.role;
      return Array.isArray(roles) ? roles : [roles];
    } catch {
      return [];
    }
  }
}
```

---

### src/app/app.module.ts (main module)

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';
import { AdminPanelComponent } from './components/admin-panel/admin-panel.component';
import { UserPanelComponent } from './components/user-panel/user-panel.component';
import { StaffPanelComponent } from './components/staff-panel/staff-panel.component';
import { ModeratorPanelComponent } from './components/moderator-panel/moderator-panel.component';

import { AuthInterceptor } from './services/auth.interceptor';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
    AdminPanelComponent,
    UserPanelComponent,
    StaffPanelComponent,
    ModeratorPanelComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule,
    FormsModule
  ],
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

---

# Summary

* Login


form posts credentials, receives JWT, stores token

* AuthInterceptor adds JWT token automatically to all HTTP requests
* `AppComponent` parses roles from token, shows role panels conditionally
* Each panel calls a role-protected API endpoint
* Unauthorized or error shows error message

---


