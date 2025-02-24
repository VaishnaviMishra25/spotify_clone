import React from 'react';

export default function App() {
  return (
    <div className="h-screen w-full flex bg-black text-white">
      {/* Sidebar */}
      <div className="w-1/4 bg-gray-900 p-5 space-y-4">
        <h1 className="text-2xl font-bold">Spotify Clone</h1>
        <nav className="space-y-2">
          <a href="#" className="block py-2 px-4 rounded-lg bg-gray-800">Home</a>
          <a href="#" className="block py-2 px-4 rounded-lg hover:bg-gray-800">Search</a>
          <a href="#" className="block py-2 px-4 rounded-lg hover:bg-gray-800">Your Library</a>
        </nav>
        <div className="pt-4">
          <h2 className="text-lg font-semibold">Playlists</h2>
          <ul className="space-y-1">
            <li className="py-1">🔥 Top Hits</li>
            <li className="py-1">🎸 Rock Classics</li>
            <li className="py-1">🎶 Chill Vibes</li>
          </ul>
        </div>
      </div>

      {/* Main Content */}
      <div className="w-3/4 p-6 overflow-y-auto">
        <h1 className="text-3xl font-bold mb-6">Good Morning</h1>
        <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
          {['Top Hits', 'Rock Classics', 'Chill Vibes', 'Workout Mix'].map((playlist) => (
            <div
              key={playlist}
              className="bg-gray-800 p-4 rounded-lg hover:bg-gray-700 cursor-pointer"
            >
              <h2 className="text-xl font-semibold">{playlist}</h2>
              <p className="text-sm text-gray-400">Playlist description here</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
