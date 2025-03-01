const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bodyParser = require('body-parser');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const bcrypt = require('bcrypt');
const rateLimit = require('express-rate-limit');

const app = express();
const port = 4000;
const SECRET_KEY = 'supersecretkey';

app.use(cors());
app.use(express.json());
app.use(bodyParser.json());

const upload = multer({ dest: 'uploads/' });

let users = [{ id: 1, username: 'admin', password: bcrypt.hashSync('password', 10), role: 'admin' }];
let playlists = [];
let songs = {};
let comments = {};
let downloads = {};

// Rate Limiting (Prevent abuse)
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
});
app.use(limiter);

// Middleware for Logging Requests
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// User Registration with Password Hashing
app.post('/register', async (req, res) => {
  const { username, password } = req.body;
  if (users.find(u => u.username === username)) {
    return res.status(400).json({ success: false, message: 'Username already exists' });
  }
  const hashedPassword = await bcrypt.hash(password, 10);
  const newUser = { id: users.length + 1, username, password: hashedPassword, role: 'user' };
  users.push(newUser);
  res.json({ success: true, message: 'User registered successfully' });
});

// User Authentication with JWT
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = users.find(u => u.username === username);
  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).json({ success: false, message: 'Invalid credentials' });
  }
  const token = jwt.sign({ id: user.id, username: user.username, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
  res.json({ success: true, token });
});

// Middleware for Verifying JWT
const verifyToken = (req, res, next) => {
  const token = req.headers['authorization'];
  if (!token) return res.status(403).json({ success: false, message: 'No token provided' });
  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) return res.status(401).json({ success: false, message: 'Unauthorized' });
    req.user = decoded;
    next();
  });
};

// Create a Playlist
app.post('/playlists', verifyToken, (req, res) => {
  const { name, description } = req.body;
  const newPlaylist = { id: playlists.length + 1, name, description, likes: 0, createdBy: req.user.username };
  playlists.push(newPlaylist);
  songs[newPlaylist.id] = [];
  res.json({ success: true, message: 'Playlist created', playlist: newPlaylist });
});

// Upload and Add a Song
app.post('/playlists/:id/songs', verifyToken, upload.single('song'), (req, res) => {
  const playlistId = parseInt(req.params.id);
  if (!songs[playlistId]) return res.status(404).json({ success: false, message: 'Playlist not found' });

  const newSong = {
    id: Date.now(),
    title: req.body.title,
    artist: req.body.artist,
    likes: 0,
    filePath: req.file.path,
    addedBy: req.user.username,
  };

  songs[playlistId].push(newSong);
  res.json({ success: true, message: 'Song added', song: newSong });
});

// Stream a Song Instead of Downloading
app.get('/playlists/:id/songs/:songId/stream', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);
  let song = songs[playlistId]?.find(s => s.id === songId);

  if (!song) return res.status(404).json({ success: false, message: 'Song not found' });

  res.setHeader('Content-Type', 'audio/mpeg');
  fs.createReadStream(song.filePath).pipe(res);
});

// Like or Dislike a Song
app.post('/playlists/:id/songs/:songId/reaction', verifyToken, (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);
  const { action } = req.body;
  let song = songs[playlistId]?.find(s => s.id === songId);

  if (!song) return res.status(404).json({ success: false, message: 'Song not found' });

  if (action === 'like') song.likes += 1;
  else if (action === 'dislike') song.likes -= 1;

  res.json({ success: true, message: `Song ${action}d`, likes: song.likes });
});

// Get All Playlists with Created By Information
app.get('/playlists', (req, res) => {
  res.json({ success: true, playlists });
});

// Search Songs and Playlists with Genre Filter
app.get('/search', (req, res) => {
  const query = req.query.q.toLowerCase();
  const genre = req.query.genre?.toLowerCase();

  let filteredPlaylists = playlists.filter(p => p.name.toLowerCase().includes(query));
  let filteredSongs = Object.values(songs).flat().filter(s => s.title.toLowerCase().includes(query));

  if (genre) {
    filteredSongs = filteredSongs.filter(s => s.genre?.toLowerCase() === genre);
  }

  res.json({ success: true, playlists: filteredPlaylists, songs: filteredSongs });
});

// Get User Profile
app.get('/profile', verifyToken, (req, res) => {
  const user = users.find(u => u.id === req.user.id);
  res.json({ success: true, user: { id: user.id, username: user.username, role: user.role } });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
