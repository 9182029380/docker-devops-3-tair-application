# Dockerizing React Hooks + Spring Boot + MySQL Full-Stack Application

## 📋 Prerequisites
- Docker installed (version 20.10+)
- Docker Compose installed (version 1.29+)
- Git installed
- Basic understanding of Docker concepts

## 🎯 Application Architecture
This application consists of three main components:
1. **Frontend**: React application (Port 3000)
2. **Backend**: Spring Boot REST API (Port 8080)
3. **Database**: MySQL (Port 3306)

---

## 📥 Step 1: Clone the Repository

```bash
git clone https://github.com/RameshMF/React-Hooks-Spring-Boot-CRUD-Full-Stack-App.git
cd React-Hooks-Spring-Boot-CRUD-Full-Stack-App
```

---

## 🐳 Step 2: Create Dockerfile for Spring Boot Backend

Create a file named `Dockerfile` in the `springboot-backend` directory:

**Path**: `springboot-backend/Dockerfile`

```dockerfile
# Use OpenJDK 17 image
FROM openjdk:17-jdk-slim

# Set working directory
WORKDIR /app

# Copy Maven wrapper and project files
COPY . .

# Build the application (uses Maven wrapper)
RUN ./mvnw clean package -DskipTests || mvn clean package -DskipTests

# Copy the built jar (if target exists)
RUN cp target/*.jar app.jar

# Expose port
EXPOSE 8080

# Environment variables
ENV SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/employee_management_system
ENV SPRING_DATASOURCE_USERNAME=root
ENV SPRING_DATASOURCE_PASSWORD=root

# Start the application
ENTRYPOINT ["java", "-jar", "app.jar"]

```

---

## 🐳 Step 3: Create Dockerfile for React Frontend

Create a file named `Dockerfile` in the `react-frontend` directory:

**Path**: `react-frontend/Dockerfile`

```dockerfile
# Multi-stage build for React
FROM node:16-alpine AS build

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Second stage - Nginx server
FROM nginx:alpine

# Copy build files to nginx
COPY --from=build /app/build /usr/share/nginx/html

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

---

## 🔧 Step 4: Create Nginx Configuration for React

Create `nginx.conf` in the `react-frontend` directory:

**Path**: `react-frontend/nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to backend
    location /api/ {
        proxy_pass http://springboot-backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

---

## 🐳 Step 5: Update React API Configuration

Update the API base URL in your React application to use relative paths or environment variables.

**Path**: `react-frontend/src/services/EmployeeService.js`

Update the base URL:
```javascript
const API_BASE_URL = process.env.REACT_APP_API_URL || "/api/v1/employees";
```

Or if it's already using a specific URL, change it to:
```javascript
const API_BASE_URL = "/api/v1/employees";
```

---

## 🐳 Step 6: Create Docker Compose File

Create `docker-compose.yml` in the root directory:

**Path**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  # MySQL Database
  mysql-db:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: employee_management_system
      MYSQL_PASSWORD: root
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - fullstack-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Spring Boot Backend
  springboot-backend:
    build:
      context: ./springboot-backend
      dockerfile: Dockerfile
    container_name: springboot-backend
    restart: always
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-db:3306/employee_management_system?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SPRING_JPA_SHOW_SQL: "true"
    depends_on:
      mysql-db:
        condition: service_healthy
    networks:
      - fullstack-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s

  # React Frontend
  react-frontend:
    build:
      context: ./react-frontend
      dockerfile: Dockerfile
    container_name: react-frontend
    restart: always
    ports:
      - "3000:80"
    depends_on:
      - springboot-backend
    networks:
      - fullstack-network

volumes:
  mysql-data:
    driver: local

networks:
  fullstack-network:
    driver: bridge
```

---

## 🗃️ Step 7: Create Database Initialization Script (Optional)

Create `init.sql` in the root directory to initialize the database:

**Path**: `init.sql`

```sql
CREATE DATABASE IF NOT EXISTS employee_management_system;
USE employee_management_system;

-- The table will be created automatically by Spring Boot JPA
-- This script is optional and can include seed data if needed

-- Example seed data (optional)
-- INSERT INTO employees (first_name, last_name, email_id) 
-- VALUES ('John', 'Doe', 'john.doe@example.com');
```

---

## 🐳 Step 8: Update Spring Boot Application Properties

Ensure your `application.properties` uses environment variables:

**Path**: `springboot-backend/src/main/resources/application.properties`

```properties
spring.application.name=springboot-backend

# Database Configuration
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/employee_management_system}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:root}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:root}

# JPA Configuration
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.hibernate.ddl-auto=${SPRING_JPA_HIBERNATE_DDL_AUTO:update}
spring.jpa.show-sql=${SPRING_JPA_SHOW_SQL:false}

# Server Configuration
server.port=8080
```

---

## 🚀 Step 9: Build and Run the Application

### Build all containers:
```bash
docker-compose build
```

### Start all services:
```bash
docker-compose up -d
```

### View logs:
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f springboot-backend
docker-compose logs -f react-frontend
docker-compose logs -f mysql-db
```

### Check container status:
```bash
docker-compose ps
```

---

## 🧪 Step 10: Access the Application

Once all containers are running:

- **React Frontend**: http://localhost:3000
- **Spring Boot API**: http://localhost:8080/api/v1/employees
- **MySQL Database**: localhost:3307 (from host machine)

---

## 🛠️ Step 11: Useful Docker Commands

### Stop all services:
```bash
docker-compose down
```

### Stop and remove volumes (clean slate):
```bash
docker-compose down -v
```

### Rebuild a specific service:
```bash
docker-compose build springboot-backend
docker-compose up -d springboot-backend
```

### Execute commands inside containers:
```bash
# Access MySQL container
docker exec -it mysql-db mysql -uroot -proot

# Access backend container
docker exec -it springboot-backend bash

# Access frontend container
docker exec -it react-frontend sh
```

### View container resource usage:
```bash
docker stats
```

---

## 🐛 Troubleshooting

### Issue 1: Backend can't connect to MySQL
**Solution**: Ensure MySQL container is healthy before backend starts
```bash
docker-compose logs mysql-db
```

### Issue 2: Port already in use
**Solution**: Change port mapping in docker-compose.yml
```yaml
ports:
  - "8081:8080"  # Use 8081 instead of 8080
```

### Issue 3: Frontend can't reach backend
**Solution**: Check network connectivity
```bash
docker network inspect react-hooks-spring-boot-crud-full-stack-app_fullstack-network
```

### Issue 4: Changes not reflecting
**Solution**: Rebuild without cache
```bash
docker-compose build --no-cache
docker-compose up -d
```

---

## 📦 Step 12: Production Optimization (Optional)

### Create production docker-compose file:

**Path**: `docker-compose.prod.yml`

```yaml
version: '3.8'

services:
  mysql-db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-prod-data:/var/lib/mysql
    networks:
      - prod-network

  springboot-backend:
    build:
      context: ./springboot-backend
      dockerfile: Dockerfile
    restart: always
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-db:3306/${DB_NAME}
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate
    depends_on:
      - mysql-db
    networks:
      - prod-network

  react-frontend:
    build:
      context: ./react-frontend
      dockerfile: Dockerfile
    restart: always
    ports:
      - "80:80"
    depends_on:
      - springboot-backend
    networks:
      - prod-network

volumes:
  mysql-prod-data:

networks:
  prod-network:
    driver: bridge
```

### Create .env file for production:
```env
DB_ROOT_PASSWORD=secure_root_password
DB_NAME=employee_management_system
DB_USER=app_user
DB_PASSWORD=secure_password
```

### Run production setup:
```bash
docker-compose -f docker-compose.prod.yml up -d
```

---

## 📊 Project Structure After Dockerization

```
React-Hooks-Spring-Boot-CRUD-Full-Stack-App/
├── springboot-backend/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── react-frontend/
│   ├── src/
│   ├── public/
│   ├── package.json
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml
├── docker-compose.prod.yml (optional)
├── init.sql
├── .env (for production)
└── README.md
```

---

## ✅ Verification Checklist

- [ ] All containers are running: `docker-compose ps`
- [ ] MySQL is accessible and database is created
- [ ] Spring Boot backend is responding at http://localhost:8080/api/v1/employees
- [ ] React frontend is accessible at http://localhost:3000
- [ ] Frontend can communicate with backend API
- [ ] CRUD operations work correctly
- [ ] Data persists after container restart

---

## 🎉 Success!

Your full-stack application is now fully dockerized and running in containers! You can now:
- Deploy to any environment with Docker support
- Scale individual services independently
- Ensure consistent development and production environments
- Share your application easily with team members

---

## 📚 Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Spring Boot with Docker](https://spring.io/guides/topicals/spring-boot-docker/)
- [Dockerizing React Apps](https://create-react-app.dev/docs/deployment/)
