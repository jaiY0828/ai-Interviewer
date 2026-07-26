# AI Interviewer

AI Interviewer is a full-stack interview preparation platform designed around realistic practice sessions, authenticated user flows, interview history, and paid access support.

The project is structured as a JavaScript monorepo with a React/Vite client and a Node.js/Express server.

## Project Overview

This application is intended to support:

- A responsive React frontend for interview setup, question flow, results, and history
- Backend APIs for authentication, interview sessions, user history, and payments
- MongoDB persistence for users, interviews, attempts, and scoring data
- Firebase authentication for secure identity handling
- Razorpay integration for subscription or payment-based access

## Tech Stack

- React
- Vite
- JavaScript
- Node.js
- Express
- MongoDB
- Firebase Authentication
- Razorpay

## Folder Structure

```text
ai interviewer/
+-- client/
|   +-- src/
|   |   +-- components/
|   |   +-- pages/
|   |   +-- services/
|   |   +-- main.jsx
|   +-- .env
+-- server/
|   +-- controllers/
|   +-- middleware/
|   +-- models/
|   +-- routes/
|   +-- services/
|   +-- public/
|   +-- .env
+-- README.md
```

Note: this repository snapshot currently contains dependency files and environment files, but the expected application source folders above are the recommended structure for the described app.

## Getting Started

Install and run the frontend:

```bash
cd client
npm install
npm run dev
```

Install and run the backend:

```bash
cd server
npm install
npm start
```

## Environment Variables

Create `client/.env` with Firebase and API configuration:

```env
VITE_API_URL=http://localhost:5000/api
VITE_FIREBASE_API_KEY=your_firebase_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your_project_id
```

Create `server/.env` with database, auth, and payment configuration:

```env
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/ai-interviewer
FIREBASE_PROJECT_ID=your_project_id
RAZORPAY_KEY_ID=your_razorpay_key_id
RAZORPAY_KEY_SECRET=your_razorpay_secret
```

## Example Code From Project Files

These examples show how the core files can be implemented in this project structure.

### Client API Service

Example: `client/src/services/interviewApi.js`

```js
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

export async function createInterviewSession(payload, token) {
  const response = await api.post("/interviews", payload, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  return response.data;
}

export async function getInterviewHistory(token) {
  const response = await api.get("/interviews/history", {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  return response.data;
}
```

### React Interview Screen

Example: `client/src/pages/InterviewSession.jsx`

```jsx
import { useState } from "react";
import { createInterviewSession } from "../services/interviewApi";

export default function InterviewSession({ userToken }) {
  const [role, setRole] = useState("Frontend Developer");
  const [level, setLevel] = useState("Intermediate");
  const [session, setSession] = useState(null);

  async function handleStart() {
    const data = await createInterviewSession(
      { role, level, questionCount: 5 },
      userToken
    );

    setSession(data);
  }

  return (
    <main>
      <h1>AI Interview Practice</h1>

      <label>
        Role
        <input value={role} onChange={(event) => setRole(event.target.value)} />
      </label>

      <label>
        Level
        <select value={level} onChange={(event) => setLevel(event.target.value)}>
          <option>Beginner</option>
          <option>Intermediate</option>
          <option>Advanced</option>
        </select>
      </label>

      <button onClick={handleStart}>Start Interview</button>

      {session && (
        <section>
          <h2>{session.role}</h2>
          <p>{session.questions.length} questions ready</p>
        </section>
      )}
    </main>
  );
}
```

### Express Server Entry

Example: `server/index.js`

```js
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import mongoose from "mongoose";
import interviewRoutes from "./routes/interviewRoutes.js";

dotenv.config();

const app = express();

app.use(cors());
app.use(express.json());
app.use("/api/interviews", interviewRoutes);

mongoose
  .connect(process.env.MONGODB_URI)
  .then(() => {
    app.listen(process.env.PORT || 5000, () => {
      console.log(`Server running on port ${process.env.PORT || 5000}`);
    });
  })
  .catch((error) => {
    console.error("MongoDB connection failed:", error);
  });
```

### Interview Model

Example: `server/models/Interview.js`

```js
import mongoose from "mongoose";

const interviewSchema = new mongoose.Schema(
  {
    userId: {
      type: String,
      required: true,
      index: true,
    },
    role: {
      type: String,
      required: true,
    },
    level: {
      type: String,
      enum: ["Beginner", "Intermediate", "Advanced"],
      default: "Intermediate",
    },
    questions: [
      {
        prompt: String,
        answer: String,
        score: Number,
        feedback: String,
      },
    ],
  },
  { timestamps: true }
);

export default mongoose.model("Interview", interviewSchema);
```

### Interview Routes

Example: `server/routes/interviewRoutes.js`

```js
import { Router } from "express";
import {
  createInterview,
  getInterviewHistory,
} from "../controllers/interviewController.js";
import { verifyFirebaseToken } from "../middleware/verifyFirebaseToken.js";

const router = Router();

router.post("/", verifyFirebaseToken, createInterview);
router.get("/history", verifyFirebaseToken, getInterviewHistory);

export default router;
```

### Interview Controller

Example: `server/controllers/interviewController.js`

```js
import Interview from "../models/Interview.js";

export async function createInterview(req, res) {
  const { role, level, questionCount = 5 } = req.body;

  const questions = Array.from({ length: questionCount }, (_, index) => ({
    prompt: `Question ${index + 1} for a ${level} ${role}`,
  }));

  const interview = await Interview.create({
    userId: req.user.uid,
    role,
    level,
    questions,
  });

  res.status(201).json(interview);
}

export async function getInterviewHistory(req, res) {
  const interviews = await Interview.find({ userId: req.user.uid })
    .sort({ createdAt: -1 })
    .limit(20);

  res.json(interviews);
}
```

### Firebase Auth Middleware

Example: `server/middleware/verifyFirebaseToken.js`

```js
import admin from "firebase-admin";

export async function verifyFirebaseToken(req, res, next) {
  const header = req.headers.authorization || "";
  const token = header.startsWith("Bearer ") ? header.slice(7) : null;

  if (!token) {
    return res.status(401).json({ message: "Missing auth token" });
  }

  try {
    req.user = await admin.auth().verifyIdToken(token);
    next();
  } catch {
    res.status(401).json({ message: "Invalid auth token" });
  }
}
```

### Razorpay Order Service

Example: `server/services/razorpayService.js`

```js
import Razorpay from "razorpay";

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID,
  key_secret: process.env.RAZORPAY_KEY_SECRET,
});

export function createPaymentOrder(amountInRupees) {
  return razorpay.orders.create({
    amount: amountInRupees * 100,
    currency: "INR",
    receipt: `receipt_${Date.now()}`,
  });
}
```

## API Overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/api/interviews` | Create a new interview session |
| `GET` | `/api/interviews/history` | Fetch the signed-in user's interview history |
| `POST` | `/api/payments/order` | Create a Razorpay payment order |

## What This Project Demonstrates

- Full-stack JavaScript architecture
- React component composition and API integration
- Express routing, controllers, middleware, and services
- Token-based authentication with Firebase
- MongoDB schema design for interview history
- Payment integration using Razorpay
- Environment-based configuration for local and production deployments

## Interview-Ready Talking Points

- The frontend and backend are separated cleanly so each can be deployed independently.
- Firebase handles identity, while the backend protects data access with verified ID tokens.
- MongoDB stores flexible interview session data, including prompts, answers, scores, and feedback.
- Razorpay support shows how the product can support paid plans or premium interview attempts.
- The structure is designed to scale from a prototype into a production-ready SaaS-style application.
