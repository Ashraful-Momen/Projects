# Lesson 2: Real-time Chat Using WebSockets

In this lesson, we'll implement a real-time chat system using WebSockets, connecting our React frontend with a Laravel backend.

## Backend (Laravel API)

Let's create the necessary components for handling real-time messaging.

### Step 1: Create Message Model and Migration

First, create a model for our messages:

```bash
php artisan make:model Message -m
```

Edit the migration file:

```php
// database/migrations/xxxx_xx_xx_create_messages_table.php
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
        Schema::create('messages', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->foreignId('room_id')->constrained()->onDelete('cascade');
            $table->text('content');
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
        Schema::dropIfExists('messages');
    }
};
```

Update the Message model:

```php
// app/Models/Message.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Message extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'room_id',
        'content',
    ];

    /**
     * Get the user that owns the message.
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the room that owns the message.
     */
    public function room()
    {
        return $this->belongsTo(Room::class);
    }
}
```

### Step 2: Create a Message Event

Create an event for broadcasting new messages:

```bash
php artisan make:event NewMessage
```

Edit the event file:

```php
// app/Events/NewMessage.php
<?php

namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NewMessage implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $message;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(Message $message)
    {
        $this->message = $message->load('user:id,name');
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PresenceChannel('room.' . $this->message->room_id);
    }
}
```

### Step 3: Create a Messages Controller

```bash
php artisan make:controller API/MessageController
```

Edit the controller file:

```php
// app/Http/Controllers/API/MessageController.php
<?php

namespace App\Http\Controllers\API;

use App\Events\NewMessage;
use App\Http\Controllers\Controller;
use App\Models\Message;
use App\Models\Room;
use Illuminate\Http\Request;

class MessageController extends Controller
{
    /**
     * Get messages for a room.
     *
     * @param  \App\Models\Room  $room
     * @return \Illuminate\Http\Response
     */
    public function index(Room $room)
    {
        // You might want to add authorization here
        
        $messages = Message::with('user:id,name')
            ->where('room_id', $room->id)
            ->orderBy('created_at', 'asc')
            ->get();
        
        return response()->json($messages);
    }

    /**
     * Store a new message.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Room  $room
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request, Room $room)
    {
        // You might want to add authorization here
        
        $validated = $request->validate([
            'content' => 'required|string|max:1000',
        ]);
        
        $message = new Message();
        $message->user_id = auth()->id();
        $message->room_id = $room->id;
        $message->content = $validated['content'];
        $message->save();
        
        // Load the user relationship for the broadcast
        $message->load('user:id,name');
        
        // Broadcast the new message
        broadcast(new NewMessage($message))->toOthers();
        
        return response()->json($message);
    }
}
```

### Step 4: Add Routes for Messages

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/rooms/{room}/messages', [MessageController::class, 'index']);
    Route::post('/rooms/{room}/messages', [MessageController::class, 'store']);
});
```

### Step 5: Update Channel Routes

If you haven't already, update your channels.php file to authorize the presence channel:

```php
// routes/channels.php
Broadcast::channel('room.{roomId}', function ($user, $roomId) {
    $room = \App\Models\Room::find($roomId);
    return $room !== null; // You can add more complex authorization logic
});
```

## Frontend (React)

Now, let's implement the chat feature in React:

### Step 1: Create a Chat Hook

Create a custom hook to manage chat messages:

```javascript
// src/hooks/useChat.js
import { useState, useEffect, useCallback } from 'react';
import echo from '../utils/echo';
import axios from '../api/axios';

export default function useChat(roomId) {
    const [messages, setMessages] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    
    // Fetch existing messages
    useEffect(() => {
        const fetchMessages = async () => {
            try {
                setLoading(true);
                const response = await axios.get(`/api/rooms/${roomId}/messages`);
                setMessages(response.data);
                setLoading(false);
            } catch (err) {
                setError('Failed to load messages');
                setLoading(false);
                console.error('Error fetching messages:', err);
            }
        };
        
        fetchMessages();
    }, [roomId]);
    
    // Listen for new messages
    useEffect(() => {
        // Join the presence channel
        const channel = echo.join(`room.${roomId}`);
        
        // Listen for new messages
        channel.listen('.new-message', (e) => {
            setMessages(prevMessages => [...prevMessages, e.message]);
        });
        
        return () => {
            echo.leave(`room.${roomId}`);
        };
    }, [roomId]);
    
    // Send a new message
    const sendMessage = useCallback(async (content) => {
        try {
            const response = await axios.post(`/api/rooms/${roomId}/messages`, {
                content
            });
            
            // Add the new message to the state (optimistic update)
            setMessages(prevMessages => [...prevMessages, response.data]);
            
            return response.data;
        } catch (err) {
            console.error('Error sending message:', err);
            throw err;
        }
    }, [roomId]);
    
    return {
        messages,
        loading,
        error,
        sendMessage
    };
}
```

### Step 2: Create Chat Components

Now, create the chat UI components:

```jsx
// src/components/ChatBox.js
import React, { useState, useRef, useEffect } from 'react';
import useChat from '../hooks/useChat';
import ChatMessage from './ChatMessage';
import ChatInput from './ChatInput';
import '../styles/Chat.css';

function ChatBox({ roomId, userId }) {
    const { messages, loading, error, sendMessage } = useChat(roomId);
    const messagesEndRef = useRef(null);
    
    // Scroll to bottom when new messages arrive
    useEffect(() => {
        if (messagesEndRef.current) {
            messagesEndRef.current.scrollIntoView({ behavior: 'smooth' });
        }
    }, [messages]);
    
    // Handle sending a new message
    const handleSendMessage = async (content) => {
        if (content.trim()) {
            try {
                await sendMessage(content);
            } catch (err) {
                console.error('Failed to send message:', err);
            }
        }
    };
    
    return (
        <div className="chat-box">
            <div className="chat-header">
                <h3>Chat</h3>
            </div>
            
            <div className="chat-messages">
                {loading ? (
                    <div className="chat-loading">Loading messages...</div>
                ) : error ? (
                    <div className="chat-error">{error}</div>
                ) : messages.length === 0 ? (
                    <div className="chat-empty">No messages yet. Start the conversation!</div>
                ) : (
                    messages.map(message => (
                        <ChatMessage 
                            key={message.id}
                            message={message}
                            isOwnMessage={message.user_id === parseInt(userId)}
                        />
                    ))
                )}
                <div ref={messagesEndRef} />
            </div>
            
            <ChatInput onSendMessage={handleSendMessage} />
        </div>
    );
}

export default ChatBox;
```

```jsx
// src/components/ChatMessage.js
import React from 'react';

function ChatMessage({ message, isOwnMessage }) {
    const formattedTime = new Date(message.created_at).toLocaleTimeString([], {
        hour: '2-digit',
        minute: '2-digit'
    });
    
    return (
        <div className={`chat-message ${isOwnMessage ? 'own-message' : 'other-message'}`}>
            <div className="message-content">
                <div className="message-header">
                    <span className="message-sender">{message.user.name}</span>
                    <span className="message-time">{formattedTime}</span>
                </div>
                <div className="message-text">{message.content}</div>
            </div>
        </div>
    );
}

export default ChatMessage;
```

```jsx
// src/components/ChatInput.js
import React, { useState } from 'react';

function ChatInput({ onSendMessage }) {
    const [message, setMessage] = useState('');
    
    const handleSubmit = (e) => {
        e.preventDefault();
        if (message.trim
