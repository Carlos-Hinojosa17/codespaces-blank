# 🧩 Arquitectura Base del Sistema Web con Microservicios

Este documento describe la **estructura base** de un sistema con varios microservicios, diseñado para un **sistema web** (usado por trabajadores) y una **página web** (usada por clientes).  
El objetivo es lograr una arquitectura **escalable, mantenible y segura**.

---

## 🚀 Descripción General

El sistema se compone de dos partes principales:

1. **Sistema Web (interno)**  
   - Usado por trabajadores como administradores, vendedores, almacenistas, etc.  
   - Requiere autenticación mediante JWT.  
   - Permite la gestión de usuarios, productos, ventas, pedidos, almacenes, reportes y configuración.

2. **Página Web (externa)**  
   - Usada por clientes para visualizar productos y realizar pedidos o cotizaciones.  

---

## 🧱 1. Estructura Base de un Microservicio

Ejemplo de la estructura de `usuarios/`, que se replica para todos los microservicios:

```
/microservicios/
└── usuarios/
    ├── src/
    │   ├── config/
    │   ├── models/
    │   ├── controllers/
    │   ├── routes/
    │   ├── middleware/
    │   ├── services/
    │   ├── utils/
    │   ├── app.js
    │   └── server.js
    ├── tests/
    ├── package.json
    ├── .env
    ├── Dockerfile
    └── README.md
```

---

## ⚙️ 2. Explicación de Carpetas

### 📁 `src/config/`
Configuraciones generales:
- **db.js:** Conexión a la base de datos.  
- **env.js:** Variables de entorno.

### 📁 `src/models/`
Define las tablas o colecciones.

Ejemplo `Usuario.js`:
```js
import { DataTypes } from "sequelize";
import { db } from "../config/db.js";

export const Usuario = db.define("Usuario", {
  id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
  nombre: { type: DataTypes.STRING, allowNull: false },
  email: { type: DataTypes.STRING, unique: true, allowNull: false },
  password: { type: DataTypes.STRING, allowNull: false },
});
```

### 📁 `src/controllers/`
Reciben la petición y delegan la lógica al servicio.

Ejemplo `usuariosController.js`:
```js
import { usuariosService } from "../services/usuariosService.js";

export const usuariosController = {
  registrar: async (req, res) => {
    try {
      const usuario = await usuariosService.registrar(req.body);
      res.status(201).json(usuario);
    } catch (error) {
      res.status(400).json({ msg: error.message });
    }
  },
  login: async (req, res) => {
    try {
      const token = await usuariosService.login(req.body);
      res.json({ token });
    } catch (error) {
      res.status(401).json({ msg: error.message });
    }
  },
};
```

### 📁 `src/services/`
Contiene la lógica de negocio.

Ejemplo `usuariosService.js`:
```js
import { Usuario } from "../models/Usuario.js";
import { generarJWT } from "../utils/jwt.js";
import bcrypt from "bcryptjs";

export const usuariosService = {
  registrar: async ({ nombre, email, password }) => {
    const existe = await Usuario.findOne({ where: { email } });
    if (existe) throw new Error("El correo ya está registrado");
    const hash = await bcrypt.hash(password, 10);
    const nuevo = await Usuario.create({ nombre, email, password: hash });
    return nuevo;
  },

  login: async ({ email, password }) => {
    const usuario = await Usuario.findOne({ where: { email } });
    if (!usuario) throw new Error("Usuario no encontrado");
    const valido = await bcrypt.compare(password, usuario.password);
    if (!valido) throw new Error("Contraseña incorrecta");
    return generarJWT(usuario);
  },
};
```

### 📁 `src/routes/`
Define los endpoints.

```js
import { Router } from "express";
import { usuariosController } from "../controllers/usuariosController.js";

const router = Router();
router.post("/register", usuariosController.registrar);
router.post("/login", usuariosController.login);
export default router;
```

### 📁 `src/middleware/`
Manejo de autenticación y permisos.

Ejemplo `authMiddleware.js`:
```js
import jwt from "jsonwebtoken";
import dotenv from "dotenv";
dotenv.config();

export const verificarToken = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ msg: "Token requerido" });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    res.status(403).json({ msg: "Token inválido o expirado" });
  }
};
```

### 📁 `src/utils/`
Funciones auxiliares, como generación de JWT.

```js
import jwt from "jsonwebtoken";

export const generarJWT = (usuario) => {
  return jwt.sign(
    { id: usuario.id, email: usuario.email, rol: usuario.rol },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );
};
```

### 📁 `src/app.js`
Configuración principal de Express.

```js
import express from "express";
import cors from "cors";
import usuariosRoutes from "./routes/usuariosRoutes.js";

const app = express();
app.use(cors());
app.use(express.json());
app.use("/users", usuariosRoutes);
export default app;
```

### 📁 `src/server.js`
Punto de inicio.

```js
import app from "./app.js";
import { db } from "./config/db.js";

const PORT = process.env.PORT || 3001;

(async () => {
  try {
    await db.authenticate();
    await db.sync({ alter: true });
    console.log("Base de datos conectada");
    app.listen(PORT, () => console.log(`Usuarios corriendo en puerto ${PORT}`));
  } catch (error) {
    console.error("Error al iniciar:", error);
  }
})();
```

### 📁 `.env`
```
PORT=3001
DB_HOST=localhost
DB_NAME=usuarios_db
DB_USER=postgres
DB_PASS=12345
JWT_SECRET=mi_clave_secreta
```

---

## 🔁 3. Microservicios del Sistema

| Microservicio | Descripción | Puerto |
|----------------|-------------|--------|
| Usuarios | Registro, login, roles, permisos | 3001 |
| Productos | Categorías, marcas, modelos y stock | 3002 |
| Ventas | Gestión de ventas y cotizaciones | 3003 |
| Pedidos | Pedidos realizados por clientes | 3004 |
| Almacén | Control de inventario y movimientos | 3005 |
| Reportes | Estadísticas, reportes personalizados | 3006 |
| Configuración | Cambios en logo, colores, nombre, etc. | 3007 |

Cada microservicio tiene su propio `.env`, `Dockerfile` y base de datos independiente.

---

## 🌐 4. API Gateway

El **API Gateway** se ubica al frente de todos los microservicios y cumple varias funciones:

### Funciones principales:
1. **Comunicación centralizada:**  
   Redirige solicitudes a cada microservicio interno según la ruta.
2. **Autenticación global:**  
   Verifica JWT antes de permitir el acceso a rutas protegidas.
3. **Control de roles:**  
   Permite o deniega rutas dependiendo del rol del usuario.
4. **Balanceo de carga:**  
   Distribuye tráfico entre instancias si hay más de una.
5. **Registro de logs y auditorías.**
6. **Gestión de CORS, cache y rate limiting.**

Ejemplo de configuración (Node.js + Express + http-proxy-middleware):

```js
import express from "express";
import { createProxyMiddleware } from "http-proxy-middleware";

const app = express();

app.use("/usuarios", createProxyMiddleware({ target: "http://localhost:3001", changeOrigin: true }));
app.use("/productos", createProxyMiddleware({ target: "http://localhost:3002", changeOrigin: true }));
app.use("/ventas", createProxyMiddleware({ target: "http://localhost:3003", changeOrigin: true }));
app.use("/pedidos", createProxyMiddleware({ target: "http://localhost:3004", changeOrigin: true }));

app.listen(8080, () => console.log("API Gateway corriendo en puerto 8080"));
```

---

## 🐳 5. Dockerfile por Microservicio

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3001
CMD ["npm", "start"]
```

---

## ✅ 6. Beneficios

| Ventaja | Descripción |
|----------|-------------|
| **Escalabilidad** | Puedes desplegar o actualizar un microservicio sin afectar a los demás. |
| **Mantenimiento claro** | Cada servicio tiene su propio ciclo de vida. |
| **Seguridad** | JWT y roles separados. |
| **Reutilización** | Estructura estándar para todos los servicios. |
| **Flexibilidad tecnológica** | Puedes usar diferentes lenguajes o bases de datos por servicio. |

---

## 🏁 Conclusión

Esta estructura te permite construir un **sistema profesional con microservicios** para tu proyecto web.  
Puedes iniciar con los microservicios de `usuarios`, `productos` y `pedidos`, e ir añadiendo los demás conforme avances.
