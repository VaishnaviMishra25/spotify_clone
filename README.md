const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bodyParser = require('body-parser');
const multer = require('multer');
const fs = require('fs');

const app = express();
const port = 4000;
const SECRET_KEY = 'supersecretkey';

app.use(cors());
app.use(express.json());
app.use(bodyParser.json());

const upload = multer({ dest: 'uploads/' });

let users = [{ id: 1, username: 'admin', password: 'password', role: 'admin' }];
let playlists = [];
let songs = {};
let comments = {};  // Stores song comments

// Middleware for logging requests
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// User registration
app.post('/register', (req, res) => {
  const { username, password } = req.body;
  if (users.find(u => u.username === username)) {
    return res.status(400).json({ success: false, message: 'Username already exists' });
  }
  const newUser = { id: users.length + 1, username, password, role: 'user' };
  users.push(newUser);
  res.json({ success: true, message: 'User registered successfully' });
});

// User authentication (JWT)
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  const user = users.find(u => u.username === username && u.password === password);
  if (!user) {
    return res.status(401).json({ success: false, message: 'Invalid credentials' });
  }
  const token = jwt.sign({ id: user.id, username: user.username, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
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

// Create a playlist
app.post('/playlists', verifyToken, (req, res) => {
  const { name, description } = req.body;
  const newPlaylist = { id: playlists.length + 1, name, description, likes: 0 };
  playlists.push(newPlaylist);
  songs[newPlaylist.id] = [];
  res.json({ success: true, message: 'Playlist created', playlist: newPlaylist });
});

// Upload and add a song
app.post('/playlists/:id/songs', verifyToken, upload.single('song'), (req, res) => {
  const playlistId = parseInt(req.params.id);
  if (!songs[playlistId]) return res.status(404).json({ success: false, message: 'Playlist not found' });
  const newSong = { id: Date.now(), title: req.body.title, artist: req.body.artist, likes: 0, filePath: req.file.path };
  songs[playlistId].push(newSong);
  res.json({ success: true, message: 'Song added', song: newSong });
});

// Like or dislike a song
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

// Comment on a song
app.post('/playlists/:id/songs/:songId/comment', verifyToken, (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);
  const { comment } = req.body;
  if (!comments[songId]) comments[songId] = [];
  comments[songId].push({ user: req.user.username, comment });
  res.json({ success: true, message: 'Comment added', comments: comments[songId] });
});

// Download a song
app.get('/playlists/:id/songs/:songId/download', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);
  let song = songs[playlistId]?.find(s => s.id === songId);
  if (!song) return res.status(404).json({ success: false, message: 'Song not found' });
  res.download(song.filePath);
});

// Get all playlists
app.get('/playlists', (req, res) => {
  res.json({ success: true, playlists });
});

// Search functionality
app.get('/search', (req, res) => {
  const query = req.query.q.toLowerCase();
  const filteredPlaylists = playlists.filter(p => p.name.toLowerCase().includes(query));
  const filteredSongs = Object.values(songs).flat().filter(s => s.title.toLowerCase().includes(query));
  res.json({ success: true, playlists: filteredPlaylists, songs: filteredSongs });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
