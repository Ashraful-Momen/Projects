# Lesson 4: File Sharing Implementation

In this lesson, we'll implement file sharing functionality for our Google Meet clone, allowing users to upload, download, and share files within a meeting.

## Backend Implementation (Laravel API)

### Step 1: Create File Model and Migration

First, let's create a model for our files:

```bash
php artisan make:model File -m
```

Edit the migration file:

```php
// database/migrations/xxxx_xx_xx_create_files_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('files', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->foreignId('room_id')->constrained()->onDelete('cascade');
            $table->string('original_name');
            $table->string('file_name');
            $table->string('file_path');
            $table->string('mime_type');
            $table->bigInteger('size');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('files');
    }
};
```

Update the File model:

```php
// app/Models/File.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class File extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'room_id',
        'original_name',
        'file_name',
        'file_path',
        'mime_type',
        'size',
    ];

    /**
     * Get the user that owns the file.
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the room that owns the file.
     */
    public function room()
    {
        return $this->belongsTo(Room::class);
    }
}
```

### Step 2: Update Filesystem Configuration

Configure your filesystem for file storage in `config/filesystems.php`:

```php
// config/filesystems.php
'disks' => [
    // ...
========================================= Broken Part =====================================
### Step 3: Create a File Event

Create an event for broadcasting new file uploads:

```bash
php artisan make:event FileUploaded
```

Edit the event file:

```php
// app/Events/FileUploaded.php
<?php

namespace App\Events;

use App\Models\File;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class FileUploaded implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $file;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(File $file)
    {
        $this->file = $file->load('user:id,name');
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PresenceChannel('room.' . $this->file->room_id);
    }
}
```

### Step 4: Create File Controller

```php
// app/Http/Controllers/API/FileController.php
<?php

namespace App\Http\Controllers\API;

use App\Events\FileUploaded;
use App\Http\Controllers\Controller;
use App\Models\File;
use App\Models\Room;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class FileController extends Controller
{
    /**
     * Get files for a room.
     *
     * @param  \App\Models\Room  $room
     * @return \Illuminate\Http\Response
     */
    public function index(Room $room)
    {
        // You might want to add authorization here
        
        $files = File::with('user:id,name')
            ->where('room_id', $room->id)
            ->orderBy('created_at', 'desc')
            ->get();
        
        return response()->json($files);
    }

    /**
     * Upload a new file.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Room  $room
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request, Room $room)
    {
        // You might want to add authorization here
        
        $request->validate([
            'file' => 'required|file|max:20480', // 20MB max size
        ]);
        
        $uploadedFile = $request->file('file');
        $originalName = $uploadedFile->getClientOriginalName();
        
        // Generate a unique filename
        $fileName = Str::uuid() . '.' . $uploadedFile->getClientOriginalExtension();
        
        // Store the file
        $path = $uploadedFile->storeAs('files/' . $room->id, $fileName, 'meeting_files');
        
        // Create file record
        $file = new File();
        $file->user_id = auth()->id();
        $file->room_id = $room->id;
        $file->original_name = $originalName;
        $file->file_name = $fileName;
        $file->file_path = $path;
        $file->mime_type = $uploadedFile->getMimeType();
        $file->size = $uploadedFile->getSize();
        $file->save();
        
        // Load user relationship for the broadcast
        $file->load('user:id,name');
        
        // Broadcast the new file upload
        broadcast(new FileUploaded($file))->toOthers();
        
        return response()->json($file);
    }

    /**
     * Download a file.
     *
     * @param  \App\Models\File  $file
     * @return \Illuminate\Http\Response
     */
    public function download(File $file)
    {
        // You might want to add authorization here
        
        // Check if the file exists
        if (!Storage::disk('meeting_files')->exists($file->file_path)) {
            return response()->json(['message' => 'File not found'], 404);
        }
        
        // Set the appropriate headers
        $headers = [
            'Content-Type' => $file->mime_type,
            'Content-Disposition' => 'attachment; filename="' . $file->original_name . '"',
        ];
        
        // Return the file download response
        return Storage::disk('meeting_files')->download($file->file_path, $file->original_name, $headers);
    }

    /**
     * Delete a file.
     *
     * @param  \App\Models\File  $file
     * @return \Illuminate\Http\Response
     */
    public function destroy(File $file)
    {
        // You might want to add authorization here
        
        // Check if user is the owner of the file
        if ($file->user_id !== auth()->id()) {
            return response()->json(['message' => 'Unauthorized'], 403);
        }
        
        // Delete the file from storage
        if (Storage::disk('meeting_files')->exists($file->file_path)) {
            Storage::disk('meeting_files')->delete($file->file_path);
        }
        
        // Delete the file record
        $file->delete();
        
        return response()->json(['message' => 'File deleted successfully']);
    }
}
```

### Step 5: Create Routes for File Operations

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    // Existing routes...
    
    // File routes
    Route::get('/rooms/{room}/files', [FileController::class, 'index']);
    Route::post('/rooms/{room}/files', [FileController::class, 'store']);
    Route::get('/files/{file}/download', [FileController::class, 'download']);
    Route::delete('/files/{file}', [FileController::class, 'destroy']);
});
```

### Step 6: Configure Queue for Large File Processing (Optional)

For handling large file uploads, you might want to use Redis for queuing the file processing:

```php
// config/queue.php
'connections' => [
    // ...
    
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => 90,
        'block_for' => null,
    ],
],
```

Create a job for processing uploaded files:

```bash
php artisan make:job ProcessUploadedFile
```

```php
// app/Jobs/ProcessUploadedFile.php
<?php

namespace App\Jobs;

use App\Models\File;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessUploadedFile implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $file;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct(File $file)
    {
        $this->file = $file;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        // Process the file (e.g., generate thumbnails, virus scanning, etc.)
        // This is just a placeholder for any additional processing you might want to do
        
        // Example: Update file status after processing
        $this->file->update(['processed' => true]);
    }
}
```

Then update the `store` method in your FileController to dispatch the job:

```php
// In FileController.php, at the end of the store method
ProcessUploadedFile::dispatch($file);
```

## Frontend Implementation (React)

### Step 1: Create File Sharing Components

First, let's create a component for displaying and uploading files:

```jsx
// src/components/FileSharing.js
import React, { useState, useEffect } from 'react';
import { useSelector } from 'react-redux';
import axios from '../api/axios';
import echo from '../utils/echo';
import FileUploader from './FileUploader';
import FileList from './FileList';
import '../styles/FileSharing.css';

function FileSharing({ roomId }) {
    const [files, setFiles] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [uploadProgress, setUploadProgress] = useState(0);
    const [isUploading, setIsUploading] = useState(false);
    const user = useSelector(state => state.auth.user);
    
    // Fetch existing files
    useEffect(() => {
        const fetchFiles = async () => {
            try {
                setLoading(true);
                const response = await axios.get(`/api/rooms/${roomId}/files`);
                setFiles(response.data);
                setLoading(false);
            } catch (err) {
                setError('Failed to load files');
                setLoading(false);
                console.error('Error fetching files:', err);
            }
        };
        
        fetchFiles();
    }, [roomId]);
    
    // Listen for new file uploads
    useEffect(() => {
        // Join the presence channel
        const channel = echo.join(`room.${roomId}`);
        
        // Listen for new file uploads
        channel.listen('.file-uploaded', (e) => {
            setFiles(prevFiles => [e.file, ...prevFiles]);
        });
        
        return () => {
            echo.leave(`room.${roomId}`);
        };
    }, [roomId]);
    
    // Handle file upload
    const handleUpload = async (file) => {
        const formData = new FormData();
        formData.append('file', file);
        
        setIsUploading(true);
        setUploadProgress(0);
        
        try {
            const response = await axios.post(`/api/rooms/${roomId}/files`, formData, {
                headers: {
                    'Content-Type': 'multipart/form-data',
                },
                onUploadProgress: progressEvent => {
                    const percentCompleted = Math.round(
                        (progressEvent.loaded * 100) / progressEvent.total
                    );
                    setUploadProgress(percentCompleted);
                },
            });
            
            // Add the new file to the list (optimistic update)
            setFiles(prevFiles => [response.data, ...prevFiles]);
            setIsUploading(false);
            
            return response.data;
        } catch (err) {
            console.error('Error uploading file:', err);
            setIsUploading(false);
            setError('Failed to upload file');
            throw err;
        }
    };
    
    // Handle file download
    const handleDownload = async (fileId) => {
        try {
            // Trigger file download
            window.open(`/api/files/${fileId}/download`, '_blank');
        } catch (err) {
            console.error('Error downloading file:', err);
            setError('Failed to download file');
        }
    };
    
    // Handle file deletion
    const handleDelete = async (fileId) => {
        try {
            await axios.delete(`/api/files/${fileId}`);
            
            // Remove the file from the list
            setFiles(prevFiles => prevFiles.filter(file => file.id !== fileId));
        } catch (err) {
            console.error('Error deleting file:', err);
            setError('Failed to delete file');
        }
    };
    
    return (
        <div className="file-sharing">
            <div className="file-sharing-header">
                <h3>Shared Files</h3>
            </div>
            
            <FileUploader 
                onUpload={handleUpload}
                progress={uploadProgress}
                isUploading={isUploading}
            />
            
            {error && <div className="file-error">{error}</div>}
            
            <FileList
                files={files}
                loading={loading}
                currentUserId={user?.id}
                onDownload={handleDownload}
                onDelete={handleDelete}
            />
        </div>
    );
}

export default FileSharing;
```

### Step 2: Create File Uploader Component

```jsx
// src/components/FileUploader.js
import React, { useState, useRef } from 'react';

function FileUploader({ onUpload, progress, isUploading }) {
    const [selectedFile, setSelectedFile] = useState(null);
    const fileInputRef = useRef(null);
    
    const handleFileChange = (e) => {
        if (e.target.files && e.target.files.length > 0) {
            setSelectedFile(e.target.files[0]);
        }
    };
    
    const handleUpload = async () => {
        if (!selectedFile) return;
        
        try {
            await onUpload(selectedFile);
            setSelectedFile(null);
            
            // Reset file input
            if (fileInputRef.current) {
                fileInputRef.current.value = '';
            }
        } catch (err) {
            console.error('Upload failed:', err);
        }
    };
    
    const formatFileSize = (bytes) => {
        if (bytes < 1024) return bytes + ' B';
        else if (bytes < 1048576) return (bytes / 1024).toFixed(1) + ' KB';
        else return (bytes / 1048576).toFixed(1) + ' MB';
    };
    
    return (
        <div className="file-uploader">
            <div className="file-input-container">
                <input
                    type="file"
                    ref={fileInputRef}
                    onChange={handleFileChange}
                    disabled={isUploading}
                    className="file-input"
                    id="file-input"
                />
                <label htmlFor="file-input" className="file-input-label">
                    Choose File
                </label>
                
                <button
                    onClick={handleUpload}
                    disabled={!selectedFile || isUploading}
                    className="upload-button"
                >
                    {isUploading ? 'Uploading...' : 'Upload'}
                </button>
            </div>
            
            {selectedFile && (
                <div className="selected-file">
                    <div className="file-name">{selectedFile.name}</div>
                    <div className="file-size">{formatFileSize(selectedFile.size)}</div>
                </div>
            )}
            
            {isUploading && (
                <div className="upload-progress">
                    <div className="progress-bar">
                        <div
                            className="progress-bar-fill"
                            style={{ width: `${progress}%` }}
                        ></div>
                    </div>
                    <div className="progress-text">{progress}%</div>
                </div>
            )}
        </div>
    );
}

export default FileUploader;
```

### Step 3: Create File List Component

```jsx
// src/components/FileList.js
import React from 'react';

function FileList({ files, loading, currentUserId, onDownload, onDelete }) {
    const formatFileSize = (bytes) => {
        if (bytes < 1024) return bytes + ' B';
        else if (bytes < 1048576) return (bytes / 1024).toFixed(1) + ' KB';
        else return (bytes / 1048576).toFixed(1) + ' MB';
    };
    
    const formatDate = (dateString) => {
        const date = new Date(dateString);
        return date.toLocaleString();
    };
    
    // Get file icon based on mime type
    const getFileIcon = (mimeType) => {
        if (mimeType.startsWith('image/')) {
            return '🖼️';
        } else if (mimeType.startsWith('video/')) {
            return '🎬';
        } else if (mimeType.startsWith('audio/')) {
            return '🎵';
        } else if (mimeType.includes('pdf')) {
            return '📄';
        } else if (mimeType.includes('word') || mimeType.includes('document')) {
            return '📝';
        } else if (mimeType.includes('excel') || mimeType.includes('spreadsheet')) {
            return '📊';
        } else if (mimeType.includes('powerpoint') || mimeType.includes('presentation')) {
            return '📽️';
        } else if (mimeType.includes('zip') || mimeType.includes('compressed')) {
            return '🗜️';
        } else {
            return '📁';
        }
    };
    
    return (
        <div className="file-list">
            {loading ? (
                <div className="file-loading">Loading files...</div>
            ) : files.length === 0 ? (
                <div className="file-empty">No files shared yet</div>
            ) : (
                <div className="file-items">
                    {files.map(file => (
                        <div key={file.id} className="file-item">
                            <div className="file-icon">
                                {getFileIcon(file.mime_type)}
                            </div>
                            
                            <div className="file-details">
                                <div className="file-name" title={file.original_name}>
                                    {file.original_name}
                                </div>
                                
                                <div className="file-meta">
                                    <span className="file-size">{formatFileSize(file.size)}</span>
                                    <span className="file-uploader">by {file.user.name}</span>
                                    <span className="file-date">{formatDate(file.created_at)}</span>
                                </div>
                            </div>
                            
                            <div className="file-actions">
                                <button
                                    onClick={() => onDownload(file.id)}
                                    className="file-download-btn"
                                    title="Download"
                                >
                                    ⬇️
                                </button>
                                
                                {file.user_id === currentUserId && (
                                    <button
                                        onClick={() => onDelete(file.id)}
                                        className="file-delete-btn"
                                        title="Delete"
                                    >
                                        🗑️
                                    </button>
                                )}
                            </div>
                        </div>
                    ))}
                </div>
            )}
        </div>
    );
}

export default FileList;
```

### Step 4: Add CSS Styles

Create a CSS file for the file sharing components:

```css
/* src/styles/FileSharing.css */
.file-sharing {
    display: flex;
    flex-direction: column;
    height: 100%;
    background-color: #f5f5f5;
    border-radius: 8px;
    overflow: hidden;
}

.file-sharing-header {
    padding: 12px 16px;
    background-color: #4285F4;
    color: white;
    font-weight: 500;
}

.file-sharing-header h3 {
    margin: 0;
}

/* File Uploader Styles */
.file-uploader {
    padding: 16px;
    background-color: white;
    border-bottom: 1px solid #e0e0e0;
}

.file-input-container {
    display: flex;
    align-items: center;
    gap: 10px;
}

.file-input {
    display: none;
}

.file-input-label {
    padding: 8px 16px;
    background-color: #f0f0f0;
    color: #333;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.file-input-label:hover {
    background-color: #e0e0e0;
}

.upload-button {
    padding: 8px 16px;
    background-color: #4285F4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.upload-button:hover {
    background-color: #3367d6;
}

.upload-button:disabled {
    background-color: #9e9e9e;
    cursor: not-allowed;
}

.selected-file {
    margin-top: 12px;
    padding: 8px;
    background-color: #f9f9f9;
    border-radius: 4px;
    display: flex;
    justify-content: space-between;
}

.upload-progress {
    margin-top: 12px;
}

.progress-bar {
    height: 8px;
    background-color: #e0e0e0;
    border-radius: 4px;
    overflow: hidden;
}

.progress-bar-fill {
    height: 100%;
    background-color: #4285F4;
    transition: width 0.3s;
}

.progress-text {
    text-align: center;
    margin-top: 4px;
    font-size: 12px;
    color: #757575;
}

/* File List Styles */
.file-list {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
}

.file-loading,
.file-empty,
.file-error {
    text-align: center;
    padding: 20px;
    color: #757575;
}

.file-error {
    color: #f44336;
}

.file-items {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.file-item {
    display: flex;
    align-items: center;
    background-color: white;
    border-radius: 8px;
    padding: 12px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.file-icon {
    font-size: 24px;
    margin-right: 12px;
}

.file-details {
    flex: 1;
    min-width: 0; /* Allow text to truncate */
}

.file-name {
    font-weight: 500;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    margin-bottom: 4px;
}

.file-meta {
    display: flex;
    font-size: 12px;
    color: #757575;
    gap: 8px;
}

.file-actions {
    display: flex;
    gap: 8px;
}

.file-download-btn,
.file-delete-btn {
    background: none;
    border: none;
    font-size: 20px;
    cursor: pointer;
    padding: 4px;
    border-radius: 50%;
    transition: background-color 0.3s;
}

.file-download-btn:hover,
.file-delete-btn:hover {
    background-color: #f0f0f0;
}

.file-delete-btn:hover {
    color: #f44336;
}

/* Responsive styles */
@media (max-width: 768px) {
    .file-meta {
        flex-direction: column;
        gap: 2px;
    }
    
    .file-actions {
        flex-direction: column;
    }
}
```

### Step 5: Update Room Component to Include File Sharing

Now, let's update the Room component to include the file sharing feature:

```jsx
// src/pages/Room.js (update)
import React, { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import axios from '../api/axios';
import VideoGrid from '../components/VideoGrid';
import ChatBox from '../components/ChatBox';
import FileSharing from '../components/FileSharing';

function Room() {
    const { roomId } = useParams();
    const [roomData, setRoomData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [activeTab, setActiveTab] = useState('chat'); // 'chat' or 'files'
    const userId = localStorage.getItem('userId') || '1'; // Get actual user ID from auth
    
    useEffect(() => {
        const fetchRoomData = async () => {
            try {
                const response = await axios.get(`/api/rooms/${roomId}`);
                setRoomData(response.data);
                setLoading(false);
            } catch (error) {
                console.error('Error fetching room data:', error);
                setLoading(false);
            }
        };
        
        fetchRoomData();
    }, [roomId]);
    
    if (loading) {
        return <div className="loading">Loading...</div>;
    }
    
    if (!roomData) {
        return <div className="error">Room not found</div>;
    }
    
    return (
        <div className="room-container">
            <div className="room-header">
                <h1>{roomData.name}</h1>
                
                <div className="room-tabs">
                    <button 
                        className={`tab-button ${activeTab === 'chat' ? 'active' : ''}`}
                        onClick={() => setActiveTab('chat')}
                    >
                        Chat
                    </button>
                    <button 
                        className={`tab-button ${activeTab === 'files' ? 'active' : ''}`}
                        onClick={() => setActiveTab('files')}
                    >
                        Files
                    </button>
                </div>
            </div>
            
            <div className="room-content">
                <div className="video-container">
                    <VideoGrid roomId={roomId} userId={userId} />
                </div>
                
                <div className="sidebar-container">
                    {activeTab === 'chat' ? (
                        <ChatBox roomId={roomId} userId={userId} />
                    ) : (
                        <FileSharing roomId={roomId} />
                    )}
                </div>
            </div>
        </div>
    );
}

export default Room;
```

Add these styles to your room CSS:

```css
/* Add to src/styles/Room.css */
.room-tabs {
    display: flex;
    gap: 10px;
}

.tab-button {
    padding: 8px 16px;
    background-color: #f0f0f0;
    border: none;
    border-radius: 4px 4px 0 0;
    cursor: pointer;
}

.tab-button.active {
    background-color: #4285F4;
    color: white;
}

.sidebar-container {
    width: 30%;
    min-width: 300px;
    border-left: 1px solid #e0e0e0;
    height: 100%;
}

@media (max-width: 768px) {
    .room-content {
        flex-direction: column;
    }
    
    .video-container {
        width: 100%;
        height: 60vh;
    }
    
    .sidebar-container {
        width: 100%;
        height: 40vh;
        border-left: none;
        border-top: 1px solid #e0e0e0;
    }
}
```

## Redis Integration for File Processing

For better performance with file uploads, let's use Redis for caching and queuing:

### Step 1: Configure Redis in Laravel

Update your `.env` file:

```
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
QUEUE_CONNECTION=redis
```

### Step 2: Set Up Docker Configuration for Redis

Update your docker-compose.yml to include Redis:

```yaml
# docker-compose.yml
services:
  # ... other services
  
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app_network

volumes:
  # ... other volumes
  redis_data:
```

### Step 3: Run Queue Worker

To process file uploads in the background:

```bash
# In a separate terminal or as a supervisor process
php artisan queue:work redis
```

## Common Issues and Solutions

1. **File Size Limitations**: PHP and server configurations often limit file uploads. Check `php.ini` settings:
   - `upload_max_filesize`
   - `post_max_size`
   - `max_execution_time`

2. **Storage Permissions**: Ensure storage directories have proper write permissions.

3. **Queue Configuration**: If file uploads seem slow, make sure your queue worker is running.

4. **Browser Compatibility**: Some older browsers may not fully support the File API features.

5. **Network Issues**: Large file uploads can fail over unstable connections. Consider implementing chunked uploads for very large files.

## Best Practices

1. **File Validation**: Always validate files on both frontend and backend (type, size, etc.).

2. **Security Checks**: Consider implementing virus scanning for uploaded files.

3. **Progress Indicators**: Always show upload progress for better user experience.

4. **Responsive Design**: Make file sharing UI responsive for mobile devices.

5. **Error Handling**: Provide clear error messages when uploads fail.

6. **Cleanup Procedures**: Implement procedures to clean up unused or expired files.

With this implementation, you have a comprehensive file sharing system integrated into your Google Meet clone. Users can upload, download, and manage files within each meeting room, enhancing collaboration possibilities.

## Next Steps

- Add previews for image and PDF files
- Implement drag-and-drop file uploads
- Add file search functionality
- Implement file version history
- Add collaborative document editing
- Implement file encryption for sensitive documents
# Lesson 4: File Sharing Implementation

In this lesson, we'll implement file sharing functionality for our Google Meet clone, allowing users to upload, download, and share files within a meeting.

## Backend Implementation (Laravel API)

### Step 1: Create File Model and Migration

First, let's create a model for our files:

```bash
php artisan make:model File -m
```

Edit the migration file:

```php
// database/migrations/xxxx_xx_xx_create_files_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('files', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->foreignId('room_id')->constrained()->onDelete('cascade');
            $table->string('original_name');
            $table->string('file_name');
            $table->string('file_path');
            $table->string('mime_type');
            $table->bigInteger('size');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('files');
    }
};
```

Update the File model:

```php
// app/Models/File.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class File extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'room_id',
        'original_name',
        'file_name',
        'file_path',
        'mime_type',
        'size',
    ];

    /**
     * Get the user that owns the file.
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the room that owns the file.
     */
    public function room()
    {
        return $this->belongsTo(Room::class);
    }
}
```

### Step 2: Update Filesystem Configuration

Configure your filesystem for file storage in `config/filesystems.php`:

```php
// config/filesystems.php
'disks' => [
    // ...
    
    'meeting_files' => [
        'driver' => 'local',
        'root' => storage_path('app/meeting_files'),
        'url' => env('APP_URL').'/storage/meeting_files',
        'visibility' => 'private',
    ],
