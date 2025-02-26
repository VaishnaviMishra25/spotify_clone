const express = require('express');
const cors = require('cors');

const app = express();
const port = 4000;

app.use(cors());
app.use(express.json());

// Sample playlists and songs data
let playlists = [
  { id: 1, name: 'Top Hits', description: 'Trending songs of the week' },
  { id: 2, name: 'Rock Classics', description: 'Legendary rock anthems' },
  { id: 3, name: 'Chill Vibes', description: 'Relaxing tunes to unwind' },
  { id: 4, name: 'Workout Mix', description: 'High-energy tracks for exercise' }
];

let songs = {
  1: [
    { id: 101, title: 'Song A', artist: 'Artist 1' },
    { id: 102, title: 'Song B', artist: 'Artist 2' }
  ],
  2: [
    { id: 103, title: 'Rock Anthem', artist: 'Band 1' },
    { id: 104, title: 'Classic Tune', artist: 'Band 2' }
  ],
  3: [
    { id: 105, title: 'Chill Track 1', artist: 'DJ Relax' },
    { id: 106, title: 'Smooth Jazz', artist: 'Sax Player' }
  ],
  4: [
    { id: 107, title: 'Pump It Up', artist: 'Gym Beats' },
    { id: 108, title: 'Energy Boost', artist: 'DJ Power' }
  ]
};

// Get all playlists
app.get('/playlists', (req, res) => {
  res.json({ success: true, data: playlists });
});

// Get songs for a specific playlist
app.get('/playlists/:id/songs', (req, res) => {
  const playlistId = parseInt(req.params.id);

  if (isNaN(playlistId)) {
    return res.status(400).json({ success: false, message: 'Invalid playlist ID format' });
  }

  const playlistSongs = songs[playlistId];

  if (!playlistSongs) {
    return res.status(404).json({ success: false, message: 'Playlist not found' });
  }

  res.json({ success: true, data: playlistSongs });
});

// Add a new playlist
app.post('/playlists', (req, res) => {
  const { name, description } = req.body;

  if (!name || !description) {
    return res.status(400).json({ success: false, message: 'Name and description are required' });
  }

  const newPlaylist = {
    id: playlists.length + 1,
    name,
    description
  };

  playlists.push(newPlaylist);
  songs[newPlaylist.id] = [];

  res.status(201).json({ success: true, data: newPlaylist });
});

// Delete a playlist
app.delete('/playlists/:id', (req, res) => {
  const playlistId = parseInt(req.params.id);

  playlists = playlists.filter(p => p.id !== playlistId);
  delete songs[playlistId];

  res.json({ success: true, message: 'Playlist deleted successfully' });
});

// Add a song to a playlist
app.post('/playlists/:id/songs', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const { title, artist } = req.body;

  if (!title || !artist) {
    return res.status(400).json({ success: false, message: 'Title and artist are required' });
  }

  if (!songs[playlistId]) {
    return res.status(404).json({ success: false, message: 'Playlist not found' });
  }

  const newSong = {
    id: Date.now(),
    title,
    artist
  };

  songs[playlistId].push(newSong);
  res.status(201).json({ success: true, data: newSong });
});

// Delete a song from a playlist
app.delete('/playlists/:id/songs/:songId', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const songId = parseInt(req.params.songId);

  if (!songs[playlistId]) {
    return res.status(404).json({ success: false, message: 'Playlist not found' });
  }

  songs[playlistId] = songs[playlistId].filter(song => song.id !== songId);
  res.json({ success: true, message: 'Song deleted successfully' });
});

// Update a playlist's name/description
app.put('/playlists/:id', (req, res) => {
  const playlistId = parseInt(req.params.id);
  const { name, description } = req.body;

  let playlist = playlists.find(p => p.id === playlistId);
  if (!playlist) {
    return res.status(404).json({ success: false, message: 'Playlist not found' });
  }

  if (name) playlist.name = name;
  if (description) playlist.description = description;

  res.json({ success: true, data: playlist });
});

// Handle invalid JSON errors
app.use((err, req, res, next) => {
  if (err instanceof SyntaxError && err.status === 400 && 'body' in err) {
    return res.status(400).json({ success: false, message: 'Invalid JSON format' });
  }
  next();
});

// Handle invalid routes
app.use((req, res) => {
  res.status(404).json({ success: false, message: 'Endpoint not found' });
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
