# Kinoman - Online Cinema Backend

> [Русская версия](./README.ru.md)

A production-ready REST API backend for a full-featured online cinema platform. Built with **NestJS** and **MongoDB**, it powers a movie streaming service with user authentication, role-based access control, movie catalog management, a rating system, favorites, and Telegram bot notifications.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Authentication & Authorization](#authentication--authorization)
- [Key Features](#key-features)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Scripts](#scripts)
- [Project Structure](#project-structure)
- [Contact](#contact)

---

## Overview

**Kinoman Backend** is the server-side of an online cinema application. It exposes a RESTful API consumed by the frontend client, handling everything from user registration and JWT authentication to movie catalog browsing, rating management, file uploads, and real-time Telegram notifications for new movie releases.

---

## Tech Stack

| Category | Technology |
|---|---|
| Framework | NestJS 10 + TypeScript 5 |
| Database | MongoDB (via Mongoose + Typegoose) |
| Authentication | Passport.js + JWT (access + refresh tokens) |
| Password Hashing | bcryptjs (10 salt rounds) |
| File Upload | Multer + fs-extra + NestJS Serve Static |
| Telegram Bot | Telegraf 4 |
| Validation | class-validator + class-transformer |
| Testing | Jest + Supertest |
| Linting / Formatting | ESLint + Prettier |

---

## Architecture

The project follows NestJS's modular, feature-based architecture with clear separation of concerns:

```
src/
├── auth/          # Registration, login, token refresh
├── user/          # Profile, favorites, admin user CRUD
├── movie/         # Movie catalog, view counter, search
├── genre/         # Genre management, collections
├── actor/         # Actor management with movie counts
├── rating/        # Per-user movie ratings
├── file/          # File upload & static serving
├── telegram/      # Telegram bot notifications
├── config/        # MongoDB & JWT configuration
├── shared/        # Enums, constants
├── pipes/         # Custom IdValidationPipe (ObjectId check)
└── main.ts        # Bootstrap: port 4200, prefix /api, CORS
```

**Patterns used:**
- MVC per module (Controller → Service → Model)
- Dependency Injection via NestJS IoC container
- Guard-based authorization (`JwtAuthGuard`, `OnlyAdminGuard`)
- Custom decorators: `@Auth()`, `@User()`
- DTO validation with class-validator
- MongoDB aggregation pipelines (actors with movie counts)
- Upsert pattern for ratings (create-or-update)
- Mongoose `.populate()` for related documents

---

## API Reference

**Base URL:** `http://localhost:4200/api`

### Auth - `/auth`

| Method | Path | Description | Access |
|---|---|---|---|
| POST | `/register` | Register a new user | Public |
| POST | `/login` | Login and receive token pair | Public |
| POST | `/login/access-token` | Refresh access token | Public |

### Users - `/users`

| Method | Path | Description | Access |
|---|---|---|---|
| GET | `/profile` | Get own profile | User |
| PUT | `/profile` | Update own profile | User |
| GET | `/profile/favorites` | Get favorite movies | User |
| PUT | `/profile/favorites` | Toggle movie in favorites | User |
| GET | `/count` | Total user count | Admin |
| GET | `/` | List all users (search support) | Admin |
| GET | `/:id` | Get user by ID | Admin |
| PUT | `/:id` | Update user (role included) | Admin |
| DELETE | `/:id` | Delete user | Admin |

### Movies - `/movies`

| Method | Path | Description | Access |
|---|---|---|---|
| GET | `/` | List movies (search support) | Public |
| GET | `/by-slug/:slug` | Get movie by slug | Public |
| GET | `/by-actor/:actorId` | Get movies by actor | Public |
| POST | `/by-genres` | Get movies by genre IDs | Public |
| GET | `/most-popular` | Top movies by view count | Public |
| PUT | `/update-count-opened` | Increment view counter | Public |
| POST | `/` | Create empty movie template | Admin |
| GET | `/:id` | Get movie by ID | Admin |
| PUT | `/:id` | Update movie | Admin |
| DELETE | `/:id` | Delete movie | Admin |

### Genres - `/genres`

| Method | Path | Description | Access |
|---|---|---|---|
| GET | `/` | List genres (search support) | Public |
| GET | `/by-slug/:slug` | Get genre by slug | Public |
| GET | `/popular` | Popular genres | Public |
| GET | `/collections` | Genres with movie posters | Public |
| POST | `/` | Create genre | Admin |
| GET | `/:id` | Get genre by ID | Admin |
| PUT | `/:id` | Update genre | Admin |
| DELETE | `/:id` | Delete genre | Admin |

### Actors - `/actor`

| Method | Path | Description | Access |
|---|---|---|---|
| GET | `/` | List actors with movie counts | Public |
| GET | `/by-slug/:slug` | Get actor by slug | Public |
| POST | `/` | Create actor | Admin |
| GET | `/:id` | Get actor by ID | Admin |
| PUT | `/:id` | Update actor | Admin |
| DELETE | `/:id` | Delete actor | Admin |

### Ratings - `/ratings`

| Method | Path | Description | Access |
|---|---|---|---|
| GET | `/:movieId` | Get user's rating for a movie | User |
| POST | `/set-rating` | Set or update rating `{ movieId, value }` | User |

### Files - `/files`

| Method | Path | Description | Access |
|---|---|---|---|
| POST | `/?folder=name` | Upload file, returns `{ url, name }` | Admin |

---

## Database Schema

### User
```ts
{
  email: string         // unique
  password: string      // bcrypt hashed
  isAdmin: boolean      // default: false
  favorites: ObjectId[] // refs: Movie
}
```

### Movie
```ts
{
  poster: string
  bigPoster: string
  title: string
  slug: string          // unique, URL-friendly ID
  videoUrl: string
  parameters: {
    year: number
    duration: number    // minutes
    country: string
  }
  rating: number        // default: 4.0, auto-updated on ratings
  countOpened: number   // view counter
  genres: ObjectId[]    // refs: Genre
  actors: ObjectId[]    // refs: Actor
  isSendTelegram: boolean // tracks if notification was sent
}
```

### Genre
```ts
{
  name: string
  slug: string    // unique
  description: string
  icon: string    // icon name/url
}
```

### Actor
```ts
{
  name: string
  slug: string    // unique
  photo: string
}
```

### Rating
```ts
{
  userId: ObjectId    // ref: User
  movieId: ObjectId   // ref: Movie
  value: number       // user's rating value
}
```

---

## Authentication & Authorization

**Flow:**
1. User registers with email + password → password hashed via bcryptjs (10 salt rounds)
2. Login returns a **token pair**: short-lived access token (1h) + long-lived refresh token (10d)
3. Client sends `Authorization: Bearer <accessToken>` on protected requests
4. `JwtAuthGuard` validates the token via Passport JWT strategy
5. `OnlyAdminGuard` checks `user.isAdmin` for admin-only routes

**Custom decorators:**
- `@Auth(Roles.User)` - requires authenticated user
- `@Auth(Roles.Admin)` - requires admin role
- `@User('email')` - extracts specific field from the JWT payload

---

## Key Features

- **JWT Auth** - access + refresh token pair with separate expiry times
- **Role-Based Access Control** - user vs. admin guards on all sensitive endpoints
- **Movie Catalog** - full CRUD, slug-based URLs, search, view tracking
- **Genre Collections** - genres returned with representative movie posters
- **Actor Aggregation** - actors listed with their total movie count via MongoDB pipeline
- **Rating System** - per-user ratings with automatic recalculation of movie average
- **Favorites** - toggle movies in/out of user's personal list
- **File Uploads** - organized by folder, served as static assets under `/uploads`
- **Telegram Notifications** - bot sends movie poster + link to a channel when a new movie is published (production only)
- **Search** - case-insensitive regex search across movies, genres, actors, and users

---

## Getting Started

### Prerequisites

- Node.js ≥ 20.3.1
- MongoDB instance (local or Atlas)
- npm or yarn

### Installation

```bash
git clone <repository-url>
cd Full-online-cinema-be
npm install
```

### Configure Environment

Create a `.env` file in the project root:

```env
MONGO_URL=mongodb://localhost:27017/kinoman
JWT_SECRET=your-super-secret-key
TELEGRAM_TOKEN=your-telegram-bot-token
TELEGRAM_CHAT_ID=your-telegram-chat-id
NODE_ENV=development
```

### Run

```bash
# Development (hot reload)
npm run dev

# Production
npm run build
npm run prod
```

The API will be available at `http://localhost:4200/api`.

---

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `MONGO_URL` | MongoDB connection string | Yes |
| `JWT_SECRET` | Secret for signing JWTs | Yes |
| `TELEGRAM_TOKEN` | Telegram bot API token | Yes |
| `TELEGRAM_CHAT_ID` | Chat/channel ID for notifications | Yes |
| `NODE_ENV` | `development` / `production` | Yes |

---

## Scripts

| Script | Description |
|---|---|
| `npm run dev` | Development server with watch mode |
| `npm run build` | Compile TypeScript for production |
| `npm run prod` | Run compiled production build |
| `npm run debug` | Debug mode with watch |
| `npm test` | Run unit tests |
| `npm run test:watch` | Unit tests in watch mode |
| `npm run test:cov` | Tests with coverage report |
| `npm run test:e2e` | End-to-end tests |
| `npm run lint` | ESLint with auto-fix |
| `npm run format` | Prettier formatting |

---

## Project Structure

```
src/
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.module.ts
│   ├── dto/                  # auth.dto.ts
│   ├── guards/               # jwt.guard.ts, admin.guard.ts
│   ├── decorators/           # auth.decorator.ts, user.decorator.ts
│   └── strategies/           # jwt.strategy.ts
├── user/
│   ├── user.controller.ts
│   ├── user.service.ts
│   ├── user.module.ts
│   ├── user.model.ts         # Typegoose schema
│   └── dto/
├── movie/
│   ├── movie.controller.ts
│   ├── movie.service.ts
│   ├── movie.module.ts
│   └── movie.model.ts
├── genre/
├── actor/
├── rating/
│   └── rating.model.ts
├── file/
│   └── file.service.ts       # Multer + fs-extra
├── telegram/
│   └── telegram.service.ts   # Telegraf bot
├── config/
│   ├── mongo.config.ts
│   └── jwt.config.ts
├── pipes/
│   └── id.validation.pipe.ts # MongoDB ObjectId validation
└── main.ts
```

---

## Contact

**Oleg Melekh** - oleg.melekh.frontend@gmail.com
