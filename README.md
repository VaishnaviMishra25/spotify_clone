require('dotenv').config();
const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');
const multer = require('multer');
const AWS = require('aws-sdk');
const rateLimit = require('express-rate-limit');

const app = express();
const port = process.env.PORT || 4000;
const SECRET_KEY = process.env.SECRET_KEY || 'supersecretkey';
const REFRESH_SECRET_KEY = process.env.REFRESH_SECRET_KEY || 'refreshsecretkey';

// MongoDB Connection
mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
}).then(() => console.log("✅ Connected to MongoDB"))
  .catch(err => console.error("❌ MongoDB Connection Error:", err));

// AWS S3 Configuration
const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  region: process.env.AWS_REGION,
});

// Multer Storage Setup
const storage = multer.memoryStorage();
const upload = multer({
  storage,
  limits: { fileSize: 10 * 1024 * 1024 },
  fileFilter: (req, file, cb) => {
    if (file.mimetype.startsWith('audio/')) cb(null, true);
    else cb(new Error('Only audio files are allowed'), false);
  }
});

// Models
const User = require('./models/User');
const Playlist = require('./models/Playlist');
const Song = require('./models/Song');

app.use(cors());
app.use(express.json());

// Rate Limiting (Different limits for guests & authenticated users)
const guestLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 50 });
const userLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 200 });
app.use(guestLimiter);

// Middleware for JWT Verification
const verifyToken = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(403).json({ success: false, message: 'No token provided' });

  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) return res.status(401).json({ success: false, message: 'Unauthorized' });
    req.user = decoded;
    next();
  });
};

// Register User with Stronger Password Policy
app.post('/register', async (req, res) => {
  try {
    const { username, password, role } = req.body;
    if (!username || !password) return res.status(400).json({ success: false, message: 'Username and password are required' });
    if (!/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/.test(password))
      return res.status(400).json({ success: false, message: 'Password must contain at least 8 characters, an uppercase letter, a number, and a special character' });

    const existingUser = await User.findOne({ username });
    if (existingUser) return res.status(400).json({ success: false, message: 'Username already exists' });

    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = new User({ username, password: hashedPassword, role: role || 'user' });
    await newUser.save();
    res.json({ success: true, message: 'User registered successfully' });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Login with Refresh Token Support
app.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;
    if (!username || !password) return res.status(400).json({ success: false, message: 'Username and password are required' });

    const user = await User.findOne({ username });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    const token = jwt.sign({ id: user._id, username, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
    const refreshToken = jwt.sign({ id: user._id }, REFRESH_SECRET_KEY, { expiresIn: '7d' });
    res.json({ success: true, token, refreshToken });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Search for Songs in a Playlist
app.get('/playlists/:id/songs/search', verifyToken, async (req, res) => {
  try {
    const { query } = req.query;
    if (!query) return res.status(400).json({ success: false, message: 'Query parameter is required' });

    const songs = await Song.find({ playlist: req.params.id, title: { $regex: query, $options: 'i' } });
    res.json({ success: true, songs });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Update User Profile
app.put('/user/profile', verifyToken, async (req, res) => {
  try {
    const { username } = req.body;
    if (!username) return res.status(400).json({ success: false, message: 'Username is required' });

    await User.findByIdAndUpdate(req.user.id, { username });
    res.json({ success: true, message: 'Profile updated successfully' });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

app.listen(port, () => {
  console.log(`✅ Server running on port ${port}`);
});
