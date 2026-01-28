1)
User logged in (JWT token)
↓
Change Password form (Zod validation)
↓
API call with old, new, confirm
↓
Backend:
  - JWT verify
  - old password check
  - new === confirm check
↓
Password update
↓
Success / Error


2) middleware/auth.js

const jwt = require("jsonwebtoken");

module.exports = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader) {
    return res.status(401).json({ message: "No token" });
  }

  const token = authHeader.split(" ")[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.id;
    next();
  } catch {
    return res.status(401).json({ message: "Invalid token" });
  }
};

3) 📁 controllers/authController.js

const bcrypt = require("bcryptjs");
const User = require("../models/User");

exports.changePassword = async (req, res) => {
  const { oldPassword, newPassword, confirmPassword } = req.body;

  // backend validation (MANDATORY)
  if (!oldPassword || !newPassword || !confirmPassword) {
    return res.status(400).json({ message: "All fields required" });
  }

  if (newPassword !== confirmPassword) {
    return res
      .status(400)
      .json({ message: "New & confirm password not same" });
  }

  const user = await User.findById(req.userId);

  // old password check
  const isMatch = await bcrypt.compare(oldPassword, user.password);
  if (!isMatch) {
    return res.status(400).json({ message: "Old password incorrect" });
  }

  // prevent same password
  const same = await bcrypt.compare(newPassword, user.password);
  if (same) {
    return res
      .status(400)
      .json({ message: "New password must be different" });
  }

  user.password = await bcrypt.hash(newPassword, 10);
  await user.save();

  res.json({ message: "Password changed successfully" });
};


4) routes/authRoutes.js

const auth = require("../middleware/auth");

router.post("/change-password", auth, changePassword);

1) FRONTEND CODE (Redux + Zod)
1️⃣ Zod Schema

📁 schemas/changePasswordSchema.js

import { z } from "zod";

export const changePasswordSchema = z
  .object({
    oldPassword: z.string().min(1, "Old password required"),
    newPassword: z.string().min(6, "Min 6 characters"),
    confirmPassword: z.string().min(6)
  })
  .refine((data) => data.newPassword === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"]
  });

2) 📁 features/auth/authSlice.js

export const changePassword = createAsyncThunk(
  "auth/changePassword",
  async (data, { getState, rejectWithValue }) => {
    try {
      const token = getState().auth.token;

      const res = await axios.post(
        "http://localhost:5000/api/auth/change-password",
        data,
        {
          headers: {
            Authorization: `Bearer ${token}`
          }
        }
      );

      return res.data.message;
    } catch (err) {
      return rejectWithValue(err.response.data.message);
    }
  }
);


3) pages/ChangePassword.jsx

import { useDispatch } from "react-redux";
import { changePassword } from "../features/auth/authSlice";
import { changePasswordSchema } from "../schemas/changePasswordSchema";
import { useState } from "react";

export default function ChangePassword() {
  const dispatch = useDispatch();
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");

  const submit = async (e) => {
    e.preventDefault();
    setError("");
    setSuccess("");

    const formData = {
      oldPassword: e.target.old.value,
      newPassword: e.target.new.value,
      confirmPassword: e.target.confirm.value
    };

    // ZOD FRONTEND VALIDATION
    const result = changePasswordSchema.safeParse(formData);
    if (!result.success) {
      setError(result.error.errors[0].message);
      return;
    }

    const res = await dispatch(changePassword(formData));

    if (res.error) {
      setError(res.payload);
    } else {
      setSuccess("Password changed successfully");
      e.target.reset();
    }
  };

  return (
    <form onSubmit={submit}>
      <h2>Change Password</h2>

      {error && <p style={{ color: "red" }}>{error}</p>}
      {success && <p style={{ color: "green" }}>{success}</p>}

      <input name="old" type="password" placeholder="Old password" />
      <input name="new" type="password" placeholder="New password" />
      <input
        name="confirm"
        type="password"
        placeholder="Confirm new password"
      />

      <button>Change Password</button>
    </form>
  );
}