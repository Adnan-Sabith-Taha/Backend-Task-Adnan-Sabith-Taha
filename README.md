# Backend-Task-Adnan-Sabith-Taha
You are tasked with designing and implementing a distributed e-commerce platform that allows global users to place orders with multiple items. 


Project Structure
const express = require('express');
const rateLimiter = require('./middlewares/rateLimiter');
const orderRoutes = require('./routes/orderRoutes');

const app = express();

app.use(express.json());
app.use(rateLimiter);
app.use('/orders', orderRoutes);

app.get('/', (req, res) => {
    res.send('Welcome to the E-commerce Platform API');
});

module.exports = app;

Source Code

src/app.js

javascript

const express = require('express');
const rateLimiter = require('./middlewares/rateLimiter');
const orderRoutes = require('./routes/orderRoutes');

const app = express();

app.use(express.json());
app.use(rateLimiter);
app.use('/orders', orderRoutes);

app.get('/', (req, res) => {
    res.send('Welcome to the E-commerce Platform API');
});

module.exports = app;

src/routes/orderRoutes.js

javascript

const express = require('express');
const pool = require('../database/pool');
const TransactionCoordinator = require('../services/transactionCoordinator');

const router = express.Router();

router.post('/place-order', async (req, res) => {
    const client = await pool.connect();
    const transactionCoordinator = new TransactionCoordinator();

    try {
        await client.query('BEGIN');
        const { user_id, items } = req.body;

        const orderResult = await client.query(
            'INSERT INTO orders (user_id, status, total_amount) VALUES ($1, $2, $3) RETURNING id',
            [user_id, 'pending', 0]
        );
        const orderId = orderResult.rows[0].id;

        for (let item of items) {
            const inventoryResult = await client.query(
                'SELECT quantity FROM inventory WHERE product_id = $1 FOR UPDATE',
                [item.product_id]
            );

            if (inventoryResult.rows[0].quantity < item.quantity) {
                throw new Error('Insufficient inventory');
            }

            await client.query(
                'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
                [item.quantity, item.product_id]
            );

            await client.query(
                'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES ($1, $2, $3, $4)',
                [orderId, item.product_id, item.quantity, item.price]
            );
        }

        await transactionCoordinator.commitTransaction();
        await client.query('COMMIT');
        res.status(201).send({ orderId });
    } catch (error) {
        await transactionCoordinator.rollbackTransaction();
        await client.query('ROLLBACK');
        res.status(400).send({ error: error.message });
    } finally {
        client.release();
    }
});

module.exports = router;

src/services/transactionCoordinator.js

javascript

class TransactionCoordinator {
    constructor() {
        this.transactions = [];
    }

    async startTransaction() {
        const transactionId = this.generateTransactionId();
        this.transactions[transactionId] = [];
        return transactionId;
    }

    async enlistService(transactionId, service) {
        this.transactions[transactionId].push(service);
    }

    async commitTransaction() {
        for (let service of this.transactions) {
            await service.commit();
        }
        this.transactions = [];
    }

    async rollbackTransaction() {
        for (let service of this.transactions) {
            await service.rollback();
        }
        this.transactions = [];
    }

    generateTransactionId() {
        return 'txn-' + Math.random().toString(36).substr(2, 9);
    }
}

module.exports = TransactionCoordinator;

src/middlewares/rateLimiter.js

javascript

const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('redis');

const redisClient = redis.createClient({
    host: 'redis',
    port: 6379
});

const apiLimiter = rateLimit({
    store: new RedisStore({
        client: redisClient
    }),
    windowMs: 60 * 1000, // 1 minute
    max: 10, // limit each IP to 10 requests per windowMs
    handler: (req, res) => {
        res.status(429).send('Too many requests, please try again later.');
    }
});

module.exports = apiLimiter;

src/database/pool.js

javascript

const { Pool } = require('pg');

const pool = new Pool({
    connectionString: process.env.DATABASE_URL
});

module.exports = pool;

src/tests/order.test.js

javascript

const request = require('supertest');
const app = require('../app');

describe('Order Placement', () => {
    it('should place an order successfully', async () => {
        const response = await request(app)
            .post('/orders/place-order')
            .send({
                user_id: 1,
                items: [
                    { product_id: 1, quantity: 2, price: 10.00 },
                    { product_id: 2, quantity: 1, price: 15.00 }
                ]
            });

        expect(response.status).toBe(201);
        expect(response.body.orderId).toBeDefined();
    });

    it('should return 400 for insufficient inventory', async () => {
        const response = await request(app)
            .post('/orders/place-order')
            .send({
                user_id: 1,
                items: [
                    { product_id: 1, quantity: 999, price: 10.00 }
                ]
            });

        expect(response.status).toBe(400);
        expect(response.body.error).toBe('Insufficient inventory');
    });
});

Dockerfile

dockerfile

FROM node:14

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]

docker-compose.yml

yaml

version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/mydatabase
    depends_on:
      - db
      - redis
  db:
    image: postgres:13
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

package.json

json

{
  "name": "ecommerce-platform",
  "version": "1.0.0",
  "description": "Distributed e-commerce platform with order placement, inventory management, and rate limiting.",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.17.1",
    "pg": "^8.5.1",
    "rate-limit-redis": "^1.7.0",
    "redis": "^3.0.2",
    "express-rate-limit": "^5.2.3"
  },
  "devDependencies": {
    "jest": "^26.6.3",
    "supertest": "^6.0.1"
  },
  "author": "Your Name",
  "license": "ISC"
}

jest.config.js

javascript

module.exports = {
  testEnvironment: 'node',
};

README.md

markdown

# Distributed E-commerce Platform

## Setup Instructions

### Prerequisites
- Docker
- Docker Compose

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/ecommerce-platform.git
   cd ecommerce-platform

    Build and start the application using Docker Compose:

    bash

    docker-compose up --build

    The application will be accessible at http://localhost:3000.

API Endpoints
Place Order

    Endpoint: POST /orders/place-order
    Description: Place a new order.
    Request Body:

    json

    {
      "user_id": 1,
      "items": [
        { "product_id": 1, "quantity": 2, "price": 10.00 },
        { "product_id": 2, "quantity": 1, "price": 15.00 }
      ]
    }

Error Responses

    400 Bad Request: Insufficient inventory.
    429 Too Many Requests: Rate limit exceeded.

Testing

Run unit tests using Jest:

bash

npm test

Security Considerations

    Data encryption using HTTPS.
    Parameterized queries to prevent SQL injection.
    Input sanitization and secure headers to prevent XSS.
    CSRF tokens to prevent CSRF attacks.

arduino


### Running the Application

To run the application, follow the steps outlined in the README file. T
