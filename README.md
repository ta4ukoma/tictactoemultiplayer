# tictactoemultiplayer

Classical tic tac toe with auth, multipleer, chat.

My approximated plan

---

### 1. **Environment Setup and Project Initialization**

#### 1.1 Install Docker

1. Install Docker on your machine (if not already installed). Follow the instructions on the [official Docker website](https://docs.docker.com/get-docker/).
2. Install Docker Compose for easier management of multi-container applications. Instructions can also be found on the official website.

#### 1.1.1 Create new git repo opn github

1. Go to github and create new repo with README file.
2. Go to terminal and execute the command `git clone`.

#### 1.2 Project Initialization

1. Create the basic project structure:

   - `frontend/` - for the frontend (Vue 3).
   - `backend/` - for the backend (Node.js).
   - `docker/` - for Docker configurations.

2. Initialize the frontend project:

   ```bash
   ~~vue create frontend~~
   ```

   Choose the necessary options for Vue 3 (e.g., Vuex, Vue Router).

3. Initialize the backend project:
   ```bash
   mkdir backend && cd backend
   npm init -y
   npm install express bcryptjs socket.io mongoose
   ```

#### 1.3 Docker Container Setup

1. Create a `docker-compose.yml` file to configure two containers:
   - One container for MongoDB.
   - One container for Node.js backend.

```yaml
version: "3.8"
services:
  backend:
    build: ./backend
    ports:
      - "3000:3000"
    networks:
      - tic-tac-toe-network
    depends_on:
      - mongo

  mongo:
    image: mongo
    container_name: mongodb
    networks:
      - tic-tac-toe-network
    ports:
      - "27017:27017"

networks:
  tic-tac-toe-network:
    driver: bridge
```

### 2. **Backend Development (Node.js)**

#### 2.1 Backend Folder Structure

1. Create the following folder structure for the backend project:
   ```
   backend/
   ├── models/      # For MongoDB models
   ├── routes/      # For Express routes
   ├── controllers/ # For request handling
   ├── sockets/     # For WebSocket handling
   └── server.js    # Main server file
   ```

#### 2.2 MongoDB Models

1. In `models/user.js`, create the user model:

   ```javascript
   const mongoose = require("mongoose");
   const bcrypt = require("bcryptjs");

   const userSchema = new mongoose.Schema({
     email: { type: String, required: true, unique: true },
     password: { type: String, required: true },
     wins: { type: Number, default: 0 },
   });

   userSchema.pre("save", async function (next) {
     if (!this.isModified("password")) return next();
     this.password = await bcrypt.hash(this.password, 10);
   });

   userSchema.methods.comparePassword = async function (password) {
     return bcrypt.compare(password, this.password);
   };

   module.exports = mongoose.model("User", userSchema);
   ```

#### 2.3 Routes and Handlers

1. In `routes/auth.js`, create routes for registration and login:

   ```javascript
   const express = require("express");
   const User = require("../models/user");
   const jwt = require("jsonwebtoken");

   const router = express.Router();

   // Register user
   router.post("/register", async (req, res) => {
     const { email, password } = req.body;
     const user = new User({ email, password });
     await user.save();
     res.status(201).send("User registered");
   });

   // Login user
   router.post("/login", async (req, res) => {
     const { email, password } = req.body;
     const user = await User.findOne({ email });
     if (!user || !(await user.comparePassword(password))) {
       return res.status(400).send("Invalid credentials");
     }
     const token = jwt.sign({ userId: user._id }, "secret");
     res.send({ token });
   });

   module.exports = router;
   ```

#### 2.4 WebSocket for Real-Time Interaction

1. In `sockets/game.js`, set up WebSocket handling:

   ```javascript
   const socketIo = require("socket.io");

   let io;
   let rooms = {}; // Contains information about current games

   function init(server) {
     io = socketIo(server);
     io.on("connection", (socket) => {
       socket.on("joinGame", (gameId, player) => {
         // Logic for joining a game and chatting via WebSocket
       });

       socket.on("makeMove", (gameId, move) => {
         // Logic for making a move
       });

       socket.on("disconnect", () => {
         // Handling disconnect
       });
     });
   }

   module.exports = { init };
   ```

#### 2.5 Running the Server

1. In `server.js`, set up the server and database connection:

   ```javascript
   const express = require("express");
   const mongoose = require("mongoose");
   const authRoutes = require("./routes/auth");
   const gameSockets = require("./sockets/game");
   const cors = require("cors");

   const app = express();
   app.use(cors());
   app.use(express.json());
   app.use("/auth", authRoutes);

   mongoose.connect("mongodb://mongo:27017/tic-tac-toe", {
     useNewUrlParser: true,
     useUnifiedTopology: true,
   });

   const server = app.listen(3000, () => {
     console.log("Server running on port 3000");
   });

   gameSockets.init(server);
   ```

### 3. **Frontend Development (Vue 3)**

#### 3.1 Frontend Folder Structure

1. Create the following folder structure for the frontend project:
   ```
   frontend/
   ├── components/  # Vue components
   ├── views/       # Vue pages
   ├── store/       # State management (Vuex/Pinia)
   └── App.vue      # Main component
   ```

#### 3.2 Game Board Component

1. In `components/Board.vue`, create the game board component:

   ```vue
   <template>
     <div class="board">
       <div
         v-for="(cell, index) in board"
         :key="index"
         class="cell"
         @click="makeMove(index)"
       >
         {{ cell }}
       </div>
     </div>
   </template>

   <script>
   export default {
     props: ["board", "gameId"],
     methods: {
       makeMove(index) {
         this.$emit("makeMove", index);
       },
     },
   };
   </script>

   <style scoped>
   .board {
     display: grid;
     grid-template-columns: repeat(3, 100px);
   }
   .cell {
     width: 100px;
     height: 100px;
     border: 1px solid black;
     text-align: center;
   }
   </style>
   ```

#### 3.3 WebSocket Integration

1. Install `socket.io-client`:
   ```bash
   npm install socket.io-client
   ```
2. In `store/game.js`, create a Vuex/Pinia store for game state management:

   ```javascript
   import { defineStore } from "pinia";
   import io from "socket.io-client";

   export const useGameStore = defineStore("game", {
     state: () => ({
       socket: io("http://localhost:3000"),
       gameId: null,
       board: Array(9).fill(null),
     }),
     actions: {
       joinGame(gameId) {
         this.socket.emit("joinGame", gameId, "Player");
       },
       makeMove(index) {
         this.socket.emit("makeMove", this.gameId, index);
       },
     },
   });
   ```

#### 3.4 Running the Frontend

1. Start the frontend using Vue CLI:
   ```bash
   npm run serve
   ```

### 4. **Integration and Testing**

#### 4.1 Integrating Frontend and Backend

1. Set up communication between frontend and backend via API for authentication and WebSocket for gameplay.

#### 4.2 Testing

1. Test all main features: registration, login, gameplay via WebSocket, and chat.

---

This plan provides a detailed step-by-step guide for developing the application using Docker, WebSocket, and MongoDB. If you need further explanations or details about any part, feel free to ask!
