1)
backend/
│── controllers/
│   └── authController.js
│── models/
│   └── User.js
│── routes/
│   └── authRoutes.js
│── utils/
│   └── sendEmail.js
│── config/
│   └── db.js
│── server.js
│── .env

2)
npm init -y
npm install express mongoose bcryptjs jsonwebtoken nodemailer dotenv cors

3)
const mongoose = require("mongoose");

const connectDB = async () => {
  await mongoose.connect(process.env.MONGO_URI);
  console.log("MongoDB connected");
};

module.exports = connectDB;

4)
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  isVerified: { type: Boolean, default: false }
});

module.exports = mongoose.model("User", userSchema);

5)
const nodemailer = require("nodemailer");

const sendEmail = async (email, token) => {
  const transporter = nodemailer.createTransport({
    service: "gmail",
    auth: {
      user: process.env.EMAIL_USER,
      pass: process.env.EMAIL_PASS
    }
  });

  const link = `http://localhost:5000/api/auth/verify/${token}`;

  await transporter.sendMail({
    to: email,
    subject: "Verify Your Account",
    html: `
      <h3>Verify your account</h3>
      <a href="${link}">Click here to verify</a>
    `
  });
};

module.exports = sendEmail;

6)
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const sendEmail = require("../utils/sendEmail");

// REGISTER
exports.register = async (req, res) => {
  const { email, password } = req.body;

  const hashedPassword = await bcrypt.hash(password, 10);

  const user = await User.create({
    email,
    password: hashedPassword
  });

  const token = jwt.sign(
    { id: user._id },
    process.env.JWT_SECRET,
    { expiresIn: "1d" }
  );

  await sendEmail(user.email, token);

  res.json({
    message: "Verification email sent",
    email: user.email
  });
};

// VERIFY EMAIL
exports.verifyEmail = async (req, res) => {
  try {
    const decoded = jwt.verify(req.params.token, process.env.JWT_SECRET);

    await User.findByIdAndUpdate(decoded.id, {
      isVerified: true
    });

    res.redirect("http://localhost:3000/verified");
  } catch {
    res.status(400).send("Invalid or expired link");
  }
};

// RESEND EMAIL
exports.resendEmail = async (req, res) => {
  const { email } = req.body;

  const user = await User.findOne({ email });
  if (!user) return res.status(404).json({ message: "User not found" });

  const token = jwt.sign(
    { id: user._id },
    process.env.JWT_SECRET,
    { expiresIn: "1d" }
  );

  await sendEmail(user.email, token);

  res.json({ message: "Email resent" });
};

7)
const express = require("express");
const {
  register,
  verifyEmail,
  resendEmail
} = require("../controllers/authController");

const router = express.Router();

router.post("/register", register);
router.get("/verify/:token", verifyEmail);
router.post("/resend", resendEmail);

module.exports = router;


8)
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const connectDB = require("./config/db");

const app = express();
connectDB();

app.use(cors());
app.use(express.json());

app.use("/api/auth", require("./routes/authRoutes"));

app.listen(5000, () =>
  console.log("Server running on port 5000")
);



1)
frontend/
│── src/
│   ├── app/
│   │   └── store.js
│   ├── features/
│   │   └── auth/
│   │       └── authSlice.js
│   ├── pages/
│   │   ├── Register.jsx
│   │   ├── VerifyAccount.jsx
│   │   └── Verified.jsx
│   ├── App.js
│   └── main.jsx

2)
npm install @reduxjs/toolkit react-redux axios react-router-dom

3)
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "../features/auth/authSlice";

export const store = configureStore({
  reducer: {
    auth: authReducer
  }
});

4)
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";
import axios from "axios";

export const registerUser = createAsyncThunk(
  "auth/register",
  async (data) => {
    const res = await axios.post(
      "http://localhost:5000/api/auth/register",
      data
    );
    return res.data;
  }
);

export const resendEmail = createAsyncThunk(
  "auth/resend",
  async (email) => {
    await axios.post("http://localhost:5000/api/auth/resend", { email });
  }
);

const authSlice = createSlice({
  name: "auth",
  initialState: {
    email: null,
    loading: false
  },
  extraReducers: (builder) => {
    builder
      .addCase(registerUser.fulfilled, (state, action) => {
        state.email = action.payload.email;
      });
  }
});

export default authSlice.reducer;

5)
import { useDispatch } from "react-redux";
import { registerUser } from "../features/auth/authSlice";
import { useNavigate } from "react-router-dom";

export default function Register() {
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const submit = async (e) => {
    e.preventDefault();

    const email = e.target.email.value;
    const password = e.target.password.value;

    await dispatch(registerUser({ email, password }));
    navigate("/verify");
  };

  return (
    <form onSubmit={submit}>
      <input name="email" placeholder="email" />
      <input name="password" type="password" />
      <button>Register</button>
    </form>
  );
}

6)
import { useDispatch, useSelector } from "react-redux";
import { resendEmail } from "../features/auth/authSlice";

export default function VerifyAccount() {
  const dispatch = useDispatch();
  const email = useSelector((state) => state.auth.email);

  return (
    <div>
      <h2>Verify Your Account</h2>
      <p>Email sent to: <b>{email}</b></p>

      <button onClick={() => dispatch(resendEmail(email))}>
        Resend Email
      </button>
    </div>
  );
}

7)
export default function Verified() {
  return (
    <div>
      <h1>✅ Account Verified</h1>
      <p>You can now login</p>
    </div>
  );
}

8)
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Register from "./pages/Register";
import VerifyAccount from "./pages/VerifyAccount";
import Verified from "./pages/Verified";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Register />} />
        <Route path="/verify" element={<VerifyAccount />} />
        <Route path="/verified" element={<Verified />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;

9)
import React from "react";
import ReactDOM from "react-dom/client";
import { Provider } from "react-redux";
import { store } from "./app/store";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(
  <Provider store={store}>
    <App />
  </Provider>
);

10)
const authSlice = createSlice({
  name: "auth",
  initialState: {
    email: null
  },
  reducers: {
    logout: (state) => {
      state.email = null;
      localStorage.removeItem("verifyEmail");
    }
  }
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;

11)
import { logout } from "../features/auth/authSlice";
import { useDispatch } from "react-redux";
import { useNavigate } from "react-router-dom";

export default function VerifyAccount() {
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const handleLogout = () => {
    dispatch(logout());
    navigate("/"); // register / login
  };

  return (
    <>
      <button onClick={handleLogout}>Logout</button>
    </>
  );
}


12)
exports.login = async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email });
  if (!user) {
    return res.status(400).json({ message: "Invalid credentials" });
  }

  // 1️⃣ Password check
  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) {
    return res.status(400).json({ message: "Invalid credentials" });
  }

  // 2️⃣ Email verified check 🔥
  if (!user.isVerified) {
    return res.status(403).json({
      message: "Please verify your email before login"
    });
  }

  // 3️⃣ Sab theek → JWT issue
  const token = jwt.sign(
    { id: user._id },
    process.env.JWT_SECRET,
    { expiresIn: "7d" }
  );

  res.json({ token });
};
