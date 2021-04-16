#### Создаем ref и сохраняем в него поток (streem) аудио и видео данных. Предварительно запросив разрешение на допуск к ним.

```js
// index.js
const myVideo = useRef();
const [stream, setStream] = useState();

useEffect(() => {
navigator.mediaDevices.getUserMedia({ video: true, audio: true })
.then((currentStream) => {
setStream(currentStream);

myVideo.current.srcObject = currentStream;
});

```

Записываем поток в объект srcObject. В дальнейшем объект используется для вставки видео и аудио в тег `video`

```js
//  client\src\Context.js
<Grid item xs={12} md={6}>
  <Typography variant="h5" gutterBottom>
    {call.name || 'Name'}
  </Typography>
  <video playsInline ref={userVideo} autoPlay className={classes.video} />
</Grid>
```

#### Создаем сокет на сервере (используем библиотеку)

// TODO описать различие между socket.js и webSocket

```js
const server = require('http').createServer(app)
const cors = require('cors')

const io = require('socket.io')(server, {
  cors: {
    origin: '*',
    methods: ['GET', 'POST'],
  },
})

app.use(cors())
```

#### Когда соединение установлено и получен id сокета через него можно пересылать сообщения.

```js
// index.js
const io = require('socket.io')(server, {
  cors: {
    origin: '*',
    methods: ['GET', 'POST'],
  },
})

io.on('connection', (socket) => {
  socket.emit('me', socket.id)

  socket.on('disconnect', () => {
    socket.broadcast.emit('callEnded')
  })

  socket.on('callUser', ({ userToCall, signalData, from, name }) => {
    io.to(userToCall).emit('callUser', { signal: signalData, from, name })
  })

  socket.on('answerCall', (data) => {
    io.to(data.to).emit('callAccepted', data.signal)
  })
})
```

````js
// client\src\Context.js

import { io } from 'socket.io-client'
// const socket = io('http://localhost:5000')
const socket = io('https://warm-wildwood-81069.herokuapp.com')
// ```
````

#### Теперь нужно эмитить события на стороне сервера и клиента и обрабатывать их

На стороне клиента все обработчики событий удобно поместить в один файл Contex.js и передовать вместе с дополнительными параметрами в другие компоненты через контекст React.

```js
// client\src\Context.js
const SocketContext = createContext()
...
 return (
    <SocketContext.Provider value={{
      call, callAccepted, myVideo, userVideo, stream,  name, callEnded,  me,
      setName, callUser, leaveCall, answerCall,
    }}
    >
      {children}
    </SocketContext.Provider>
  );
```

#### Обмен сообщениями в сокетах между сервером и клиентом

```js
// Сервер
socket.on('answerCall', (data) => {
  io.to(data.to).emit('callAccepted', data.signal)
})
// Клиент
socket.on('callAccepted', (signal) => {
  setCallAccepted(true)
  peer.signal(signal)
})
```

#### [PEER.JS](https://peerjs.com/) library - implifies WebRTC peer-to-peer data, video, and audio calls

Как происходит обмен сообщениями между клиентами через сервер

(1) вызов: один клиент звонит другому

```js
const callUser = (id) => {
  const peer = new Peer({ initiator: true, trickle: false, stream })

  peer.on('signal', (data) => {
    // CLIENT: fired when peer has signaling data and wants to send them to the remote peer
    socket.emit('callUser', {
      userToCall: id,
      signalData: data,
      from: me,
      name,
    })

    // SERVER: 	socket.on("callUser", ({ userToCall, signalData, from, name }) => { io.to(userToCall).emit("callUser", { signal: signalData, from, name });	});

    // const  socket = io('https://warm-wildwood-81069.herokuapp.com'
  })

  peer.on('stream', (currentStream) => {
    // Received a remote video stream, which can be displayed in a video tag
    userVideo.current.srcObject = currentStream
  })

  socket.on('callAccepted', (signal) => {
    setCallAccepted(true)
    // Call this method whenever the remote peer emits a peer.on('signal') event.
    // Pass the data from 'signal' events to the remote peer and call peer.signal(data) to get connected
    peer.signal(signal)
  })

  connectionRef.current = peer
}
```

(2) ответ на вызов клиента

```js
//  client\src\Context.js
const answerCall = () => {
  setCallAccepted(true) // useState()
  // We create a new peer that will receive data
  const peer = new Peer({ initiator: false, trickle: false, stream })

  peer.on('signal', (data) => {
    // CLIENT: when peer has signaling data, give it to socket = io('https://warm-wildwood-81069.herokuapp.com' somehow
    socket.emit('answerCall', { signal: data, to: call.from })

    // 	SERVER: socket.on("answerCall", (data) => {		io.to(data.to).emit("callAccepted", data.signal)
    //  CLIENT: socket.on('callAccepted', (signal) => {  setCallAccepted(true);  peer.signal(signal) - where const peer = new Peer({ initiator: true, trickle: false, stream });
  })

  peer.on('stream', (currentStream) => {
    // Received a remote video stream, which can be displayed in a video tag
    userVideo.current.srcObject = currentStream
  })

  // Call this method whenever the remote peer emits a peer.on('signal') event.
  // Pass the data from 'signal' events to the remote peer and call peer.signal(data) to get connected
  peer.signal(call.signal)

  connectionRef.current = peer
}
```
