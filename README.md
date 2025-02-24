const express = require('express');
const cors = require('cors');

const app = express();
const port = 4000;

app.use(cors());
app.use(express.json());

// Sample playlists and songs data
const playlists = [
  { id: 1, name: 'Top Hits', description: 'Trending songs of the week' },
  { id: 2, name: 'Rock Classics', description: 'Legendary rock anthems' },
  { id: 3, name: 'Chill Vibes', description: 'Relaxing tunes to unwind' },
  { id: 4, name: 'Workout Mix', description: 'High-energy tracks for exercise' }
];

const songs = {
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
  res.json(playlists);
});

// Get songs for a specific playlist
app.get('/playlists/:id/songs', (req, res) => {
  const playlistId = req.params.id;
  const playlistSongs = songs[playlistId];

  if (playlistSongs) {
    res.json(playlistSongs);
  } else {
    res.status(404).json({ message: 'Playlist not found' });
  }
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
