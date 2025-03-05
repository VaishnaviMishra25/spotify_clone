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
const upload = multer({ storage });

// Models
const User = require('./models/User');
const Playlist = require('./models/Playlist');
const Song = require('./models/Song');

app.use(cors());
app.use(express.json());

// Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "There are too many requests from this IP, please try again later."
});
app.use(limiter);

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

// Register User
app.post('/register', async (req, res) => {
  try {
    const { username, password } = req.body;
    const existingUser = await User.findOne({ username });
    if (existingUser) return res.status(400).json({ success: false, message: 'Username already exists' });

    const hashedPassword = await bcrypt.hash(password, 10);
    const newUser = new User({ username, password: hashedPassword, role: 'user' });
    await newUser.save();
    res.json({ success: true, message: 'User registered successfully' });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Login User
app.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;
    const user = await User.findOne({ username });

    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    const token = jwt.sign({ id: user._id, username, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
    res.json({ success: true, token });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Create Playlist
app.post('/playlists', verifyToken, async (req, res) => {
  try {
    const { name, description, isPublic } = req.body;
    const newPlaylist = new Playlist({
      name, description, isPublic, createdBy: req.user.username
    });
    await newPlaylist.save();
    res.json({ success: true, message: 'Playlist created', playlist: newPlaylist });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

// Upload Song to AWS S3
app.post('/playlists/:id/songs', verifyToken, upload.single('song'), async (req, res) => {
  try {
    const playlist = await Playlist.findById(req.params.id);
    if (!playlist) return res.status(404).json({ success: false, message: 'Playlist not found' });

    const songFile = req.file;
    const fileKey = `songs/${Date.now()}_${songFile.originalname}`;

    const uploadParams = {
      Bucket: process.env.AWS_BUCKET_NAME,
      Key: fileKey,
      Body: songFile.buffer,
      ContentType: songFile.mimetype,
    };

    const uploadResult = await s3.upload(uploadParams).promise();

    const newSong = new Song({
      title: req.body.title,
      artist: req.body.artist,
      fileUrl: uploadResult.Location,
      playlist: playlist._id,
      addedBy: req.user.username,
    });

    await newSong.save();
    res.json({ success: true, message: 'Song uploaded', song: newSong });
  } catch (error) {
    res.status(500).json({ success: false, message: 'Internal server error' });
  }
});

app.listen(port, () => {
  console.log(`✅ Server running on port ${port}`);
});
