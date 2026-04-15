# 🏦 Banking API Implementation Guide (with Winston Logging)

This guide provides a comprehensive, step-by-step walkthrough to build a secure and highly observable Banking API from scratch.

---

## 🏗️ Project Architecture Overview

This project is built using **Node.js**, **Express**, and **MongoDB**. It features:

- **Authentication**: JWT-based session management with `argon2` hashing.
- **Observability**: Production-grade logging using **Winston**.
- **Security**: Granular rate-limiting using `rate-limiter-flexible`.
- **Database**: Mongoose for modeling and validation.

---

## 🛠️ Step-by-Step Implementation

### Step 1: Project Initialization

1. Initialize the project:
   ```bash
   pnpm init
   ```
2. Install dependencies:
   ```bash
   pnpm add express mongoose jsonwebtoken argon2 cors dotenv winston rate-limiter-flexible http-errors http-status-codes
   ```
3. Add a development script in `package.json`:
   ```json
   "scripts": {
     "dev": "nodemon index.js"
   }
   ```

### Step 2: Environment Setup

Create a `.env` file for your configuration:

```env
MONGO_URI=your_mongodb_uri
PORT=3000
LOG_LEVEL=info
JWT_SECRET=your_secret
JWT_REFRESH_SECRET=your_refresh_secret
JWT_EXPIRES_IN=5m
```

### Step 3: Global Winston Configuration (`config/logger.js`)

We use Winston to centralize all application logs.

```javascript
import winston from "winston";

const { combine, timestamp, printf, colorize, errors } = winston.format;

const logFormat = printf(({ level, message, timestamp, stack }) => {
  return `${timestamp} [${level}]: ${stack || message}`;
});

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  format: combine(
    timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
    errors({ stack: true }),
    logFormat,
  ),
  transports: [
    new winston.transports.Console({
      format: combine(colorize({ all: true })),
    }),
    new winston.transports.File({ filename: "logs/error.log", level: "error" }),
    new winston.transports.File({ filename: "logs/combined.log" }),
  ],
});

export default logger;
```

### Step 4: Request Logging Middleware (`middleware/logger.middleware.js`)

This middleware captures every incoming request and logs its duration.

```javascript
import logger from "../config/logger.js";

const loggerMiddleware = (req, res, next) => {
  const startTime = Date.now();

  res.on("finish", () => {
    const duration = Date.now() - startTime;
    const message = `${req.method} ${req.originalUrl} | ${res.statusCode} | ${duration}ms`;

    switch (true) {
      case res.statusCode >= 500:
        logger.error(message);
        break;
      case res.statusCode >= 400:
        logger.warn(message);
        break;
      default:
        logger.info(message);
        break;
    }
  });

  next();
};

export default loggerMiddleware;
```

### Step 5: Master Error Handling (`middleware/error.middleware.js`)

Centralizing error logging ensures that stack traces are always captured in your `error.log`.

```javascript
import { StatusCodes } from "http-status-codes";
import logger from "../config/logger.js";

const errorMiddleware = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.message || "Internal Server Error";

  if (statusCode >= 500) {
    logger.error(`${req.method} ${req.originalUrl} - ${message}`, {
      stack: err.stack,
    });
  } else {
    logger.warn(`${req.method} ${req.originalUrl} - ${message}`);
  }

  res.status(statusCode).json({ success: false, message });
};

export default errorMiddleware;
```

### Step 6: Database Connectivity (`config/db.js`)

Ensure your database connection attempts are also logged via Winston.

```javascript
import mongoose from "mongoose";
import logger from "./logger.js";

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    logger.info("MongoDB Connection Successful");
  } catch (error) {
    logger.error(`Database connection failed: ${error.message}`);
    process.exit(1);
  }
};
```

---

## 📂 Recommended File Structure

```text
.
├── config/
│   ├── db.js
│   └── logger.js
├── controllers/
│   └── auth.controller.js
├── models/
│   └── user.model.js
├── middleware/
│   ├── error.middleware.js
│   ├── logger.middleware.js
│   └── ratelimit.middleware.js
├── routes/
│   └── auth.routes.js
├── logs/ (Ignored by Git)
├── .env
├── .gitignore
├── index.js
└── package.json
```

---

## 🚦 Running the Application

1. **Prepare Logs**: Winston will automatically create the `logs/` folder if it doesn't exist.
2. **Start Dev Server**:
   ```bash
   pnpm dev
   ```
3. **Check Output**:
   - Success logs will appear in `combined.log`.
   - Error stack traces will appear in `error.log`.

---

## ✅ Best Practices Checklist

- [x] Use `pnpm` for faster, safer dependency management.
- [x] Always prefer Winston `logger.info/error` over standard `console.log`.
- [x] Keep sensitive keys in `.env` and never upload them.
- [x] Use `res.on('finish')` to log requests only _after_ theyre processed.
