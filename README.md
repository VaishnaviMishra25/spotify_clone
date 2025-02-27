const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bodyParser = require('body-parser');
const multer = require('multer');

const app = express();
const port = 4000;
const SECRET_KEY = 'supersecretkey';

app.use(cors());
app.use(express.json());
app.use(bodyParser.json());

const upload = multer({ dest: 'uploads/' });

let users = [{ id: 1, username: 'admin', password: 'password' }];
let playlists = [
  { id: 1, name: 'Top Hits', description: 'Trending songs of the week', likes: 0 },
  { id: 2, name: 'Rock Classics', description: 'Legendary rock anthems', likes: 0 }
];
let songs = {
  1: [
    { id: 101, title: 'Song A', artist: 'Artist 1', likes: 0 },
    { id: 102, title: 'Song B', artist: 'Artist 2', likes: 0 }
  ],
};

// Middleware for logging requests
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// User authentication (JWT)
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  const user = users.find(u => u.username === username && u.password === password);
  if (!user) {
    return res.status(401).json({ success: false, message: 'Invalid credentials' });
  }
  const token = jwt.sign({ id: user.id, username: user.username }, SECRET_KEY, { expiresIn: '1h' });
  res.json({ success: true, token });
});

// Middleware for verifying JWT
const verifyToken = (req, res, next) => {
  const token = req.headers['authorization'];
  if (!token) return res.status(403).json({ success: false, message: 'No token provided' });
  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) return res.status(401).json({ success: false, message: 'Unauthorized' });
    req.user = decoded;
    next();
  });
};

// Search playlists or songs
app.get('/search', (req, res) => {
  const query = req.query.q.toLowerCase();
  const filteredPlaylists = playlists.filter(p => p.name.toLowerCase().includes(query));
  const filteredSongs = Object.values(songs).flat().filter(s => s.title.toLowerCase().includes(query));
  res.json({ success: true, playlists: filteredPlaylists, songs: filteredSongs });
});

// Like a song
app.post('/playlists/:id/songs/:songId/like', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);
  let song = songs[playlistId]?.find(s => s.id === songId);
  if (!song) return res.status(404).json({ success: false, message: 'Song not found' });
  song.likes += 1;
  res.json({ success: true, message: 'Song liked', likes: song.likes });
});

// Share a playlist (generate a unique link)
app.get('/playlists/:id/share', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const playlist = playlists.find(p => p.id === playlistId);
  if (!playlist) return res.status(404).json({ success: false, message: 'Playlist not found' });
  const shareLink = `http://localhost:${port}/playlists/${playlistId}`;
  res.json({ success: true, shareLink });
});

// Upload song file
app.post('/upload', upload.single('song'), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ success: false, message: 'No file uplo
