# FoodHub — Backend Microservices

<div align="center">

![Node.js](https://img.shields.io/badge/Node.js-18-339933?style=for-the-badge&logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express.js-000000?style=for-the-badge&logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=for-the-badge&logo=stripe&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=json-web-tokens&logoColor=white)

A production-grade, cloud-native food delivery platform built on a microservices architecture — containerized with Docker and orchestrated via Kubernetes.

</div>

---

## Table of Contents

- [Architecture](#architecture)
- [Microservices](#microservices)
- [Tech Stack](#tech-stack)
- [Features](#features)
- [API Reference](#api-reference)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
                        ┌──────────────────────────────┐
                        │        NGINX Ingress          │
                        │  (API Gateway / Load Balancer) │
                        └──────────────┬───────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                            │
  ┌────────▼───────┐      ┌────────────▼────────┐     ┌───────────▼──────┐
  │  Auth Service  │      │ Restaurant Service   │     │  Order Service   │
  │   Port: 5000   │      │    Port: 5001        │     │   Port: 5002     │
  │  JWT + OAuth2  │      │  CRUD + Menu Mgmt    │     │ Orders + Stripe  │
  └──────┬─────────┘      └────────────┬─────────┘     └───────────┬──────┘
         │                             │                            │
  ┌──────▼─────────┐      ┌────────────▼──────────┐   ┌───────────▼──────┐
  │   MongoDB       │      │      MongoDB           │   │  MongoDB         │
  │ (users, tokens) │      │ (restaurants, menus)   │   │ (orders)         │
  └─────────────────┘      └───────────────────────┘   └──────────────────┘

  ┌──────────────────┐     ┌───────────────────────┐
  │ Delivery Service │     │ Notification Service   │
  │   Port: 5003     │     │    Port: 5004          │
  │ Geo Tracking +   │     │  Email (Nodemailer)    │
  │  Auto-Assign     │     │  SMS (Twilio)          │
  └───────┬──────────┘     └───────────────────────┘
          │
  ┌───────▼──────────┐
  │    MongoDB        │
  │  (deliveries)     │
  └──────────────────┘
```

Each service is independently deployable, owns its own MongoDB database, and communicates through HTTP APIs routed through the NGINX ingress controller.

---

## Microservices

| Service | Port | Responsibility |
|---|---|---|
| **Auth Service** | 5000 | Registration, login, OTP verification, Google OAuth 2.0, JWT |
| **Restaurant Service** | 5001 | Restaurant CRUD, menu management, availability, ratings |
| **Order Service** | 5002 | Order lifecycle management, Stripe payments, refunds, webhooks |
| **Delivery Service** | 5003 | Real-time delivery tracking, geospatial auto-assignment, location updates |
| **Notification Service** | 5004 | Email & SMS notifications via Nodemailer and Twilio |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 18 (Alpine) |
| Framework | Express.js |
| Database | MongoDB + Mongoose ODM |
| Authentication | JWT, Passport.js, Google OAuth 2.0 |
| Payments | Stripe (Payment Intents + Webhooks) |
| Email | Nodemailer |
| SMS | Twilio |
| Containerization | Docker |
| Orchestration | Kubernetes |
| Ingress | NGINX Ingress Controller |

---

## Features

### Authentication & Users
- Multi-role support: `user`, `restaurant_owner`, `delivery_person`, `admin`
- OTP-based email & SMS verification on registration
- Google OAuth 2.0 single sign-on
- Password reset flow with OTP
- JWT access tokens with secure rotation

### Restaurants & Menus
- Full CRUD for restaurants and menu items
- Geospatial indexing (`2dsphere`) for proximity-based restaurant discovery
- Per-item nutritional info: calories, protein, carbs, fat
- Real-time availability toggling for restaurants and individual menu items
- Opening hours management per day of week

### Orders & Payments
- Full order lifecycle: `PENDING → CONFIRMED → PREPARING → READY_FOR_PICKUP → OUT_FOR_DELIVERY → DELIVERED`
- Stripe Payment Intents with server-side webhook confirmation
- Cash on delivery support
- Automatic refund processing on cancellations
- Geospatial delivery coordinates stored on each order

### Delivery & Tracking
- Real-time GPS location updates from delivery personnel
- Proximity-based auto-assignment: nearest available courier gets the order
- Full delivery history per courier
- Geospatial indexes on both pickup and drop-off locations

---

## API Reference

### Auth Service — `/api/auth`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/register` | — | Register a new user |
| POST | `/verify-otp` | — | Verify OTP after registration |
| POST | `/login` | — | Login and receive JWT |
| POST | `/forgot-password` | — | Request password reset OTP |
| POST | `/reset-password` | — | Reset password with OTP |
| GET | `/me` | JWT | Get authenticated user profile |
| GET | `/google` | — | Initiate Google OAuth |
| GET | `/google/callback` | — | Google OAuth callback |

### Restaurant Service — `/api/restaurants` & `/api/menu-items`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/restaurants` | JWT | List all restaurants |
| GET | `/restaurants/:id` | — | Get restaurant details |
| POST | `/restaurants` | Owner | Create a restaurant |
| PUT | `/restaurants/:id` | Owner | Update restaurant |
| PATCH | `/restaurants/:id/availability` | Owner | Toggle availability |
| GET | `/restaurants/owner/my-restaurants` | Owner | Owner's restaurants |
| POST | `/menu-items` | Owner | Create menu item |
| GET | `/menu-items/restaurant/:id` | — | Get menu for a restaurant |
| PUT | `/menu-items/:id` | Owner | Update menu item |
| PATCH | `/menu-items/:id/availability` | Owner | Toggle item availability |

### Order Service — `/api/orders` & `/api/payments`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/orders` | JWT | Create an order |
| GET | `/orders` | JWT | Get user's orders |
| GET | `/orders/:id` | JWT | Get order by ID |
| PATCH | `/orders/:id/status` | JWT | Update order status |
| PUT | `/orders/:id/delivery` | JWT | Assign delivery to order |
| GET | `/orders/ready-for-pickup` | JWT | Orders ready for courier pickup |
| POST | `/payments/create-payment-intent` | JWT | Create Stripe payment intent |
| POST | `/payments/confirm-payment` | JWT | Confirm payment |
| POST | `/payments/webhook` | — | Stripe webhook handler |

### Delivery Service — `/api/deliveries`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/deliveries` | — | Create a delivery record |
| GET | `/deliveries/:id` | — | Get delivery by ID |
| GET | `/deliveries/by-order/:orderId` | — | Get delivery by order ID |
| PUT | `/deliveries/:id/status` | — | Update delivery status |
| POST | `/deliveries/:id/location` | — | Push current GPS location |
| GET | `/deliveries/:id/location` | — | Get current location |
| PUT | `/deliveries/:id/assign` | — | Manually assign courier |
| PUT | `/deliveries/:id/auto-assign` | — | Auto-assign to nearest courier |
| GET | `/deliveries/delivery-person/:id/active` | — | Active deliveries for courier |
| GET | `/deliveries/delivery-person/:id/history` | — | Delivery history for courier |

---

## Getting Started

### Prerequisites

- [Git](https://git-scm.com/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/) (for local cluster)
- [Node.js 18+](https://nodejs.org/) (optional, for running services locally without Docker)

### Clone the Repository

```bash
git clone https://github.com/IT22056320/food-delivery-system-backend.git
cd food-delivery-system-backend
```

### Run a Single Service Locally

```bash
cd services/auth-service
npm install
npm run dev
```

Repeat for any other service. Ensure each `.env` file is configured first.

---

## Environment Variables

Each service uses its own `.env` file. Copy the provided templates and fill in your values.

### Auth Service

```env
PORT=5000
MONGO_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/auth-db
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=7d
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:5000/api/auth/google/callback
NOTIFICATION_SERVICE_URL=http://localhost:5004
```

### Restaurant Service

```env
PORT=5001
MONGO_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/restaurant-db
JWT_SECRET=your_jwt_secret
```

### Order Service

```env
PORT=5002
MONGO_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/order-db
JWT_SECRET=your_jwt_secret
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

### Delivery Service

```env
PORT=5003
MONGO_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/delivery-db
JWT_SECRET=your_jwt_secret
```

### Notification Service

```env
PORT=5004
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your@email.com
EMAIL_PASS=your_app_password
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_twilio_token
TWILIO_PHONE_NUMBER=+1234567890
```

---

## Kubernetes Deployment

### 1. Start Your Cluster

```bash
minikube start
minikube addons enable ingress
```

### 2. Configure Kubernetes Secrets

```bash
cp k8s/auth-service/auth-secrets.yaml.template k8s/auth-service/auth-secrets.yaml
cp k8s/order-service/order-secrets.yaml.template k8s/order-service/order-secrets.yaml
cp k8s/restaurant-service/restaurant-secrets.yaml.template k8s/restaurant-service/restaurant-secrets.yaml
cp k8s/delivery-service/delivery-secrets.yaml.template k8s/delivery-service/delivery-secrets.yaml
cp k8s/notification-service/notification-secrets.yaml.template k8s/notification-service/notification-secrets.yaml
```

Edit each file with your real credentials before applying.

### 3. Deploy All Services

```bash
kubectl apply -f k8s/
```

Or deploy individually:

```bash
kubectl apply -f k8s/auth-service/
kubectl apply -f k8s/restaurant-service/
kubectl apply -f k8s/order-service/
kubectl apply -f k8s/delivery-service/
kubectl apply -f k8s/notification-service/
```

### 4. Apply Ingress

```bash
kubectl apply -f k8s/restaurant-service/ingress.yaml
```

### 5. Verify Everything is Running

```bash
kubectl get pods
kubectl get services
kubectl get ingress
```

### 6. Health Check

```bash
INGRESS=$(kubectl get ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')

curl http://$INGRESS/api/auth/health
curl http://$INGRESS/api/restaurants/health
curl http://$INGRESS/api/orders/health
curl http://$INGRESS/api/deliveries/health
curl http://$INGRESS/api/notifications/health
```

---

## Troubleshooting

**Pods not starting**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Ingress not routing traffic**
```bash
kubectl get events
kubectl describe ingress
```

**Rebuilding a service after code changes**
```bash
kubectl delete deployment <service-name>
kubectl apply -f k8s/<service-name>/
```

**Common issues**
- Ensure all `*-secrets.yaml` files exist with valid values before applying.
- Verify your MongoDB Atlas cluster allows connections from your Kubernetes cluster's IP.
- For Stripe webhooks locally, use the [Stripe CLI](https://stripe.com/docs/stripe-cli) to forward events.

---

## Related Repository

[FoodHub Frontend](https://github.com/IT22056320/food-delivery-system-frontend) — Next.js 15 customer, restaurant owner, delivery, and admin dashboards.

---

<div align="center">
Built with Node.js · Express · MongoDB · Docker · Kubernetes
</div>
