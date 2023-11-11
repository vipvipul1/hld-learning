---


---

<h2 id="mvp-requirements">MVP Requirements</h2>
<h3 id="v0">V0</h3>
<ol>
<li>User can login.</li>
<li>User can choose to play against another online User/Computer.</li>
<li>User can choose to play as White/Black and difficulty level.</li>
<li>Each move can have a timer.</li>
<li>System should assign another opposite color User(random)/Computer to the User.</li>
<li>User should be able to move his piece from source to a valid destination position.</li>
<li>User should be able to view history of moves in a live game.</li>
<li>User can choose to cancel a live game.</li>
<li>System should assign only 1 live game to a User.</li>
</ol>
<h3 id="v1">V1</h3>
<ol>
<li>A User can invite another User by searching with email/username.</li>
<li>Users may be able to see all valid positions they can move a piece to.</li>
<li>Users may undo their move.</li>
<li>Users can history of games played.</li>
</ol>
<h2 id="design-goals">Design Goals</h2>
<blockquote>
<p>PACELC Theorem</p>
</blockquote>
<p>3 Options:-</p>
<ol>
<li>High Consistency, Low Availability</li>
<li>Eventual Consistency, High Availability</li>
<li>Low Consistency (Potential Data Loss), Highest Availability</li>
</ol>
<p>The website is being played by millions including International Players. So High Availability is needed. But during a game, we can’t afford to have data loss. Means, moves must be reflected in the board eventually. Stale reads for the other user is okay for a few seconds. So, I prefer <strong>Eventual Consistency, High Availability</strong>.</p>
<p>Here, stale read may be:-</p>
<ul>
<li>User1 made a move and User2 receives it, but it is still showing “sending” to User1 or vice versa.</li>
<li>User is not able to see his history of moves immediately after a move.</li>
</ul>
<p>But if there is no partition, we want to have <strong>Low Latency</strong> with Eventual Consistency so that Users can interact with each other in very less wait time.</p>
<h2 id="scale-estimation">Scale Estimation</h2>
<blockquote>
<p>Points to focus on:-</p>
</blockquote>
<ul>
<li>Read/Write Heavy.</li>
<li>Data Size to identify the need of Sharding.</li>
<li>QPS for average/peak time.</li>
</ul>
<blockquote>
<p>Tables:-</p>
</blockquote>
<ol>
<li>User - id <strong>long</strong>, username <strong>string</strong>, name <strong>string</strong>, email <strong>string</strong>, password <strong>string</strong>, createdAt <strong>timestamp</strong></li>
<li>Game - id <strong>long</strong>, userId1 <strong>User</strong>, userId2 <strong>User</strong>, startTime <strong>datetime</strong>, endTime <strong>datetime</strong>, status <strong>int</strong>,  winnerUserId <strong>User</strong></li>
<li>GameMoves - id <strong>long</strong>, gameId <strong>Game</strong>, srcRow <strong>int</strong>, srcCol <strong>int</strong>, destRow <strong>int</strong>, destCol <strong>int</strong>, moveByUserId <strong>User</strong>, createdAt <strong>timestamp</strong>, pieceId <strong>Piece</strong></li>
<li>MoveKill - gameMovesId <strong>long</strong>, pieceId <strong>Piece</strong></li>
<li>Piece - id <strong>long</strong>, piece <strong>string</strong>, color <strong>string</strong></li>
</ol>
<p>MAU - 20M<br>
DAU - 5M</p>
<p>Writes will mainly be of - create user, start game, make move, end game.</p>
<p>create user 				-&gt; 8+30+30+50+20+8=146 bytes (User table entry)<br>
<strong>total Users					-&gt; 146*1B=146gb</strong></p>
<p>start/end game 			-&gt; 8+8+8+8+8+4+8 = 52 bytes (Game table entry)<br>
1 move 						-&gt; 8+8+4+4+4+4+8+8+8 = 56 bytes (GameMoves entry)<br>
1 kill in a game			-&gt; 8+8 = 16 bytes (MoveKill entry)<br>
Total moves/game 	-&gt; 56x100(moves) = 5600 bytes<br>
Total kills/game			-&gt; 16x20(pieces/game) = 6400 bytes<br>
Games/day/user 		-&gt; (5600+6400)x20(games/day) = 12000x20 = 2.4x10^5 bytes<br>
<strong>Games/day					-&gt; 2.4x10^5 x 5M = 1.2tb</strong><br>
<strong>Games/month			-&gt; 2.4x10^5 x 20M = 4.8tb</strong></p>
<blockquote>
<p>Observations</p>
</blockquote>
<ol>
<li>Users’ data is not huge. So, no sharding for Users’ data. Game data is very huge. So, sharding is must for game data.</li>
<li>System is both read &amp; write heavy because if User is active then he is definitely playing. So, he is making moves and receiving opponents’ moves.</li>
</ol>
<h2 id="system-design">System Design</h2>
<h4 id="apis">APIs</h4>
<ol>
<li>createUser(name, phone, email, password)</li>
<li>startGame(userId, pieceColor, difficultyLevel)</li>
<li>movePiece(srcRow, srcCol, destRow, destCol)</li>
<li>endGame(userId, gameId)</li>
</ol>
<p>For storing Users’ data we will create a global Users db where each User info and his games history are stored. Games history will only have a mapping of (userId, gameId). All game related data will be stored in separate Games shard.<br>
For storing Game data, we will shard Games db based on gameId.</p>
<ul>
<li>When 2 Users start a game, a random gameUUID will be created at client side and sent to server.</li>
<li>Entry will be made in both Users db and Games db. A 2-way web-socket connection will be established between both the users connected to the same web server. This web server will also be sharded based on gameId and client will connect to this web server via Consistent Hashing on client side.</li>
<li>When a movePiece(…) request is sent by a sender, it will be first received by the web-socket server. This server will send write request to a MakeMoves app server. This MakeMoves server will then send request to Games shard via Consistent Hashing. Upon SUCCESS, the MakeMoves send back response to web-socket server. Web-scoket server will then update the receiver with the new move. Then the web-socket server will update the sender and timer will start for both the Users.</li>
<li>Here, web-socket server and db are stateful whereas app-server is stateless.</li>
<li>If there is any network partition between sender will receiver, then retry mechanism in client side will ensure that the move is sent to the server again. But, the same gameUUID will be sent again.</li>
<li>If it was saved in the 1st try but response wasn’t received due to network partition then there won’t be any new entry for the retry and SUCCESS will be returned to both sender and receiver. Otherwise, entry will be made in Games shard. In this way we can maintain the Idempotent nature of the moves.</li>
<li>We can use Master-Slave architecture for both Users and Games shard for backup with Eventual Consistency.</li>
</ul>

