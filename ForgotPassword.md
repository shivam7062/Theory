1)terminal
npm init -y
npm install express mongoose bcrypt jsonwebtoken nodemailer cors dotenv


2).env
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/authDB
JWT_SECRET=resetSecret123
EMAIL_USER=yourgmail@gmail.com
EMAIL_PASS=your_gmail_app_password
CLIENT_URL=http://localhost:3000


3)models
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true }
});

module.exports = mongoose.model("User", userSchema);


4)backend/server.js
const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");
const nodemailer = require("nodemailer");
const cors = require("cors");
require("dotenv").config();

const User = require("./models/User");

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log("MongoDB Connected"))
  .catch(err => console.log(err));

/* EMAIL CONFIG */
const transporter = nodemailer.createTransport({
  service: "gmail",
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASS,
  },
});

/* FORGOT PASSWORD */
app.post("/forgot-password", async (req, res) => {
  const { email } = req.body;

  const user = await User.findOne({ email });
  if (!user) {
    return res.status(400).json({ message: "Email not found" });
  }

  const token = jwt.sign(
    { id: user._id },
    process.env.JWT_SECRET,
    { expiresIn: "15m" }
  );

  const resetLink = `${process.env.CLIENT_URL}/reset-password?token=${token}`;

  await transporter.sendMail({
    to: email,
    subject: "Reset Your Password",
    html: `
      <h3>Password Reset</h3>
      <p>Click below link to reset password</p>
      <a href="${resetLink}">${resetLink}</a>
    `
  });

  res.json({ message: "Reset link sent to email" });
});

/* RESET PASSWORD */
app.post("/reset-password", async (req, res) => {
  const { token, password } = req.body;

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const hashedPassword = await bcrypt.hash(password, 10);

    await User.findByIdAndUpdate(decoded.id, {
      password: hashedPassword
    });

    res.json({ message: "Password reset successful" });
  } catch (err) {
    res.status(400).json({ message: "Invalid or expired token" });
  }
});

app.listen(process.env.PORT, () =>
  console.log(`Server running on port ${process.env.PORT}`)
);



1)src/App.js
import { BrowserRouter, Routes, Route } from "react-router-dom";
import ForgotPassword from "./pages/ForgotPassword";
import ResetPassword from "./pages/ResetPassword";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<ForgotPassword />} />
        <Route path="/reset-password" element={<ResetPassword />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;


2)src/pages/ForgotPassword.jsx
import { useState } from "react";

export default function ForgotPassword() {
  const [email, setEmail] = useState("");
  const [msg, setMsg] = useState("");

  const submitHandler = async () => {
    if (!email) {
      setMsg("Please enter a valid email");
      return;
    }

    const res = await fetch("http://localhost:5000/forgot-password", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email }),
    });

    const data = await res.json();
    setMsg(data.message);
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-xl shadow w-96">
        <h2 className="text-2xl font-bold text-center mb-4">
          Reset Your Password
        </h2>

        <input
          type="email"
          placeholder="Email Address"
          className="w-full border p-2 rounded mb-4"
          onChange={(e) => setEmail(e.target.value)}
        />

        <button
          onClick={submitHandler}
          className="w-full bg-blue-600 text-white py-2 rounded"
        >
          Submit
        </button>

        {msg && (
          <p className="text-center text-sm text-red-500 mt-4">
            {msg}
          </p>
        )}

        <p className="text-center mt-4 text-sm">
          <a href="/login" className="text-blue-600">
            Back to Log in
          </a>
        </p>
      </div>
    </div>
  );
}

3)📄 src/pages/ResetPassword.jsx
import { useState } from "react";
import { useSearchParams } from "react-router-dom";

export default function ResetPassword() {
  const [password, setPassword] = useState("");
  const [msg, setMsg] = useState("");
  const [params] = useSearchParams();

  const token = params.get("token");

  const resetHandler = async () => {
    if (!password) {
      setMsg("Password required");
      return;
    }

    const res = await fetch("http://localhost:5000/reset-password", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ token, password }),
    });

    const data = await res.json();
    setMsg(data.message);
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-xl shadow w-96">
        <h2 className="text-2xl font-bold text-center mb-4">
          Reset Your Password
        </h2>

        <input
          type="password"
          placeholder="New Password"
          className="w-full border p-2 rounded mb-4"
          onChange={(e) => setPassword(e.target.value)}
        />

        <button
          onClick={resetHandler}
          className="w-full bg-green-600 text-white py-2 rounded"
        >
          Reset Password
        </button>

        {msg && (
          <p className="text-center text-sm text-red-500 mt-4">
            {msg}
          </p>
        )}
      </div>
    </div>
  );
}
