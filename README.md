# quickdesk-odo
unzip QuickDesk_Initial_Setup.zip
cd QuickDesk
git init
git add .
git commit -m "Initial commit: QuickDesk hackathon setup"
git remote add origin <your-github-url>
git push -u origin main
// server/index.js
const express = require("express");
const mongoose = require("mongoose");
const dotenv = require("dotenv");
const cors = require("cors");
const authRoutes = require("./routes/auth");
const ticketRoutes = require("./routes/tickets");
const userRoutes = require("./routes/users");
const categoryRoutes = require("./routes/categories");
const commentRoutes = require("./routes/comments");

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

// Routes
app.use("/api/auth", authRoutes);
app.use("/api/tickets", ticketRoutes);
app.use("/api/users", userRoutes);
app.use("/api/categories", categoryRoutes);
app.use("/api/comments", commentRoutes);

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => {
    app.listen(5000, () => console.log("Server running on http://localhost:5000"));
  })
  .catch(err => console.log(err));

// --- MODELS, AUTH ROUTES, MIDDLEWARE OMITTED FOR BREVITY ---

// --- TICKET ROUTES ---
// routes/tickets.js
const express = require("express");
const Ticket = require("../models/Ticket");
const { auth, roleCheck } = require("../middleware/auth");
const router = express.Router();

// Create a ticket (End User)
router.post("/", auth, roleCheck(["enduser"]), async (req, res) => {
  try {
    const { subject, description, category } = req.body;
    const ticket = await Ticket.create({
      subject,
      description,
      category,
      createdBy: req.user.id,
    });
    res.status(201).json(ticket);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Get tickets (role based)
router.get("/", auth, async (req, res) => {
  try {
    let tickets;
    if (req.user.role === "admin") {
      tickets = await Ticket.find().populate("category createdBy assignedTo");
    } else if (req.user.role === "agent") {
      tickets = await Ticket.find({ $or: [{ assignedTo: req.user.id }, { status: "Open" }] }).populate("category createdBy assignedTo");
    } else {
      tickets = await Ticket.find({ createdBy: req.user.id }).populate("category assignedTo");
    }
    res.json(tickets);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Update ticket (assign/status) - Agent/Admin
router.patch("/:id", auth, roleCheck(["agent", "admin"]), async (req, res) => {
  try {
    const { status, assignedTo } = req.body;
    const update = {};
    if (status) update.status = status;
    if (assignedTo) update.assignedTo = assignedTo;

    const updated = await Ticket.findByIdAndUpdate(req.params.id, update, { new: true });
    res.json(updated);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Delete ticket (Admin only)
router.delete("/:id", auth, roleCheck(["admin"]), async (req, res) => {
  try {
    await Ticket.findByIdAndDelete(req.params.id);
    res.json({ message: "Ticket deleted" });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = router;
